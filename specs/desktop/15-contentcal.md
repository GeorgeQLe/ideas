# ContentCal Desktop — 5-Framework Comparison

## Desktop Rationale

ContentCal (social media content planner and scheduler) benefits substantially from a desktop version:

- **Local media library management:** Manage large collections of images, videos, and design assets locally without uploading everything to the cloud first. Thumbnail generation and preview happen instantly on local files.
- **Drag-and-drop content calendar:** Native drag-and-drop for rearranging posts on a visual calendar is dramatically smoother in a desktop app than a browser — no accidental page scrolls, no dropped events.
- **Offline content creation:** Draft posts, build content calendars, and organize media libraries without internet. Queue everything for publishing when back online.
- **Native image/video editing integration:** Open assets directly in Photoshop, Figma, Canva desktop, or Final Cut Pro via system file associations. Round-trip editing with automatic re-import on save.
- **Multi-account dashboard:** Manage 10-50+ social accounts across platforms in a persistent, always-open workspace without browser tab fatigue or session timeouts.

---

## Desktop-Specific Features

- **Local media vault:** Index and thumbnail local image/video directories; tag and organize without uploading
- **Drag-and-drop calendar:** Native DnD for scheduling posts across a week/month grid with smooth animations
- **External editor round-trip:** Double-click an asset to open in Photoshop/Figma; detect file save and re-import automatically
- **Bulk media import:** Drop a folder of assets to batch-import with auto-tagging (EXIF data, AI content detection)
- **Global hotkey capture:** Cmd+Shift+C to quick-draft a post idea from anywhere with clipboard content pre-filled
- **System tray scheduling status:** Badge showing upcoming posts, publishing status, and failed post alerts
- **Multi-window workspace:** Detach calendar, media library, and analytics into separate windows across monitors
- **Offline queue:** Draft and schedule posts offline; auto-publish when connection is restored
- **Native notifications:** Alerts for publishing failures, engagement milestones, and optimal posting time reminders
- **Template library:** Local storage of reusable post templates with variable placeholders for rapid content creation

---

## Shared Packages

### core (Rust)

- **Content scheduler:** Time-zone-aware scheduling engine with optimal posting time suggestions based on historical engagement data
- **Media processor:** Thumbnail generation (image-rs), video frame extraction (ffmpeg bindings), EXIF metadata reading, image resizing per platform
- **Template engine:** Post template parsing with variable substitution, hashtag management, and character counting per platform
- **Calendar logic:** Recurring post series, content pillar categorization, gap detection in publishing schedule, holiday awareness
- **AI content assistant:** Prompt builder for caption generation, hashtag suggestions, and content repurposing across platforms
- **Platform validator:** Enforce platform-specific constraints (character limits, image dimensions, video lengths, hashtag counts)
- **Analytics aggregator:** Normalize engagement metrics across platforms into unified scoring for content performance comparison

### api-client (Rust)

- **Social platform APIs:** OAuth2 connections to Instagram, Twitter/X, LinkedIn, Facebook, TikTok, Threads for publishing and analytics
- **Cloud sync:** Bidirectional sync of content calendars and post drafts across devices and team members
- **Media upload:** Chunked upload of images/videos to social platforms with retry and progress tracking
- **Analytics fetcher:** Pull engagement metrics (likes, shares, comments, reach) from each platform's analytics API
- **Offline queue:** Queue scheduled posts and API calls; replay with deduplication on reconnect

### data (SQLite)

- **Schema:** accounts, posts (id, content, platform, scheduled_at, status, media_ids[]), media_assets, templates, analytics, content_pillars, sync_queue
- **Local-first:** All drafts, schedules, and media references stored locally; cloud sync for team collaboration
- **FTS5 index:** Full-text search across post content, captions, hashtags, and template names
- **Media catalog:** File path references with cached thumbnails, dimensions, and format metadata

---

## Framework Implementations

### Electron

**Architecture:** Main process manages media file indexing, thumbnail generation (sharp), external editor file watching (chokidar), and social API connections. Renderer (React) provides the calendar UI, media browser, and post editor.

**Tech stack:**
- Renderer: React 18, FullCalendar (calendar grid), react-beautiful-dnd (drag-and-drop), TailwindCSS, Zustand
- Main: sharp (image processing), chokidar (file watching), fluent-ffmpeg (video thumbnails), better-sqlite3
- Database: better-sqlite3 with Drizzle ORM
- Media: sharp for thumbnails, exifr for metadata extraction

**Native module needs:** sharp, better-sqlite3, fluent-ffmpeg (requires ffmpeg binary)

**Bundle size:** ~190-260 MB (Chromium + sharp + ffmpeg binary)

**Memory:** ~300-500 MB (higher with large media grids and calendar views loaded)

**Pros:**
- FullCalendar is the most feature-rich calendar component available — drag-and-drop scheduling, recurring events, multi-view
- react-beautiful-dnd provides polished drag-and-drop with accessibility and smooth animations
- Rich media preview capabilities with HTML5 video player and image rendering
- Maximum code reuse from ContentCal SaaS web application

**Cons:**
- Heavy bundle includes ffmpeg binary (~40 MB) for video processing
- Memory usage escalates quickly with large media grids (hundreds of thumbnails in DOM)
- Drag-and-drop across calendar views can feel sluggish compared to native implementations
- External editor round-trip watching (chokidar) adds complexity in the main process
- Multiple social platform OAuth sessions compete for cookie storage in Chromium

### Tauri

**Architecture:** Rust backend handles media processing (image-rs, ffmpeg-next), file system watching (notify crate), and social API OAuth flows. Svelte frontend renders the calendar and media browser.

**Tech stack:**
- Frontend: Svelte 5, custom calendar component (or svelte-fullcalendar), TailwindCSS
- Backend: image (thumbnail generation), ffmpeg-next (video processing), notify (file watcher), rusqlite, reqwest, tokio
- Media: image-rs for thumbnails, kamadak-exif for metadata

**Plugin needs:** tauri-plugin-dialog, tauri-plugin-fs, tauri-plugin-notification, tauri-plugin-drag (native DnD)

**Bundle size:** ~10-18 MB (+ optional ffmpeg binary ~40 MB, or use system ffmpeg)

**Memory:** ~60-120 MB (efficient thumbnail caching)

**Pros:**
- Rust media processing (image-rs) is significantly faster than sharp for batch thumbnail generation
- Tiny base bundle — ffmpeg can be detected from system PATH or lazy-downloaded
- File system watching via notify crate is efficient and cross-platform
- Low memory footprint allows comfortable background operation for scheduled publishing

**Cons:**
- Calendar component ecosystem in Svelte is less mature than React's FullCalendar
- Building a polished drag-and-drop calendar from scratch requires significant frontend effort (3-4 extra weeks)
- Media grid rendering in webview may struggle with 500+ thumbnails without careful virtualization
- OAuth2 social platform flows through Tauri's webview require careful redirect handling
- Video preview playback depends on system webview codec support which varies across OS versions

### Flutter Desktop

**Architecture:** Custom calendar widget with GestureDetector-based drag-and-drop. Media grid with cached network images. Platform channels for file watching and external editor integration.

**Tech stack:**
- UI: Flutter 3.x, table_calendar (calendar), Riverpod, cached_network_image
- Database: drift
- Media: flutter_rust_bridge → image-rs for thumbnails, video_player for previews
- File: desktop_drop, file_picker

**Plugin needs:** window_manager, tray_manager, local_notifier, desktop_drop, url_launcher

**Bundle size:** ~25-35 MB

**Memory:** ~130-220 MB (media grid rendering)

**Pros:**
- Custom calendar widget can be designed from scratch with beautiful animations and transitions
- Gesture-based drag-and-drop feels natural and responsive on desktop
- Could extend to a mobile companion app for on-the-go content scheduling
- Hot reload accelerates iteration on the complex calendar UI layout

**Cons:**
- No FullCalendar equivalent — calendar must be built from scratch or heavily customized from table_calendar
- Video preview support on desktop is inconsistent across platforms
- Image rendering in Flutter desktop is less memory-efficient than native solutions for large media grids
- External editor integration (file watching, re-import) requires platform-specific channel implementations

### Swift/SwiftUI (macOS)

**Architecture:** SwiftUI views with native NSCalendarView inspiration. PHPickerViewController for media selection. FSEvents for file watching. NSWorkspace for external editor launch.

**Tech stack:**
- UI: SwiftUI, native drag-and-drop (onDrag/onDrop modifiers), LazyVGrid for media browser
- Data: SwiftData with CloudKit sync
- Media: AVFoundation (video thumbnails), Core Image (image processing), Photos framework
- File: FSEvents for file system monitoring
- Social: URLSession for API calls

**Bundle size:** ~8-14 MB

**Memory:** ~40-80 MB

**Pros:**
- **Best drag-and-drop experience** — SwiftUI's native DnD with UTType support is seamless
- AVFoundation provides the fastest, most reliable video thumbnail generation on macOS
- Core Image processing leverages GPU acceleration for batch image operations
- NSWorkspace.open() with file coordination enables smooth external editor round-trips
- Native Share Sheet integration for quick content ideas from any app
- Smallest memory footprint for browsing large media libraries (lazy image loading via LazyVGrid)

**Cons:**
- macOS only — social media managers often use Windows, making cross-platform critical
- No equivalent to FullCalendar — must build calendar grid in SwiftUI from scratch
- iCloud sync limits team collaboration features compared to custom backend
- Smaller ecosystem of social media platform SDKs for Swift compared to JavaScript/Python
- App Store sandboxing restricts external editor integration and local file access

### Kotlin Multiplatform

**Architecture:** Compose Multiplatform calendar UI. AWT for drag-and-drop interop. Ktor for social API OAuth flows. Kotlin Image library or JVM ImageIO for media processing.

**Tech stack:**
- UI: Compose Desktop, Material 3, custom calendar composable
- Database: SQLDelight
- Media: Java ImageIO, Xuggler/JavaCV (video processing)
- HTTP: Ktor client with OAuth2 plugin
- File: java.nio.file.WatchService

**Bundle size:** ~50-70 MB (+JRE)

**Memory:** ~200-350 MB (JVM + media processing overhead)

**Pros:**
- Ktor OAuth2 support simplifies social platform authentication flows
- java.nio.file.WatchService provides reliable cross-platform file watching
- Could share scheduling logic with an Android companion app
- Kotlin coroutines handle concurrent multi-platform API calls cleanly

**Cons:**
- Compose Desktop drag-and-drop support is limited and non-native feeling
- JVM image/video processing is slower and more memory-hungry than native alternatives
- Calendar component must be built entirely from scratch in Compose — significant effort
- Heavyweight footprint is inappropriate for an always-running social media tool
- Media grid scrolling performance in Compose lags behind Flutter and web solutions

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 7 | 10 | 10 | 9 | 11 |
| Bundle size | 230 MB | 15 MB | 30 MB | 11 MB | 62 MB |
| Memory usage | 400 MB | 90 MB | 175 MB | 60 MB | 280 MB |
| Startup time | 2.6s | 0.8s | 1.3s | 0.4s | 2.2s |
| Native feel | 5/10 | 6/10 | 6/10 | 10/10 | 4/10 |
| Offline capability | 8/10 | 9/10 | 7/10 | 8/10 | 7/10 |
| Media handling | 9/10 | 7/10 | 6/10 | 9/10 | 5/10 |
| Cross-platform | 9/10 | 9/10 | 9/10 | 2/10 | 8/10 |
| Best fit score | 8/10 | 6/10 | 6/10 | 7/10 | 4/10 |

---

## Recommended Framework

**Electron** is the best fit for ContentCal Desktop.

**Rationale:** ContentCal's core UX is a visual drag-and-drop calendar with rich media previews — FullCalendar and react-beautiful-dnd are the best-in-class libraries for this exact use case. Building an equivalent calendar experience from scratch in Tauri, Flutter, or Compose would add 3-4 weeks of development time and still not match FullCalendar's polish. Social media managers are not developer power users who notice bundle size; they care about the visual experience and workflow efficiency.

**Runner-up:** Swift/SwiftUI for a Mac-only version. Native drag-and-drop and AVFoundation media handling are superior, but the cross-platform requirement is non-negotiable for social media teams using mixed OS environments.

---

## Monetization (Desktop)

- **Free tier:** Up to 3 social accounts, 30 scheduled posts/month, local media library, basic calendar
- **Pro ($24/mo or $199/yr):** Unlimited accounts, unlimited scheduling, AI caption generation, analytics dashboard, multi-platform publishing
- **Team ($12/user/mo):** Shared content calendars, approval workflows, role-based permissions, brand asset library
- **Agency ($29/user/mo):** Multi-brand workspaces, client reporting, white-label exports, API access
- **Distribution:** Direct download for all platforms; Mac App Store for discoverability among non-technical users

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | Social account OAuth connection (Instagram, Twitter/X, LinkedIn), post composer with platform-specific previews |
| 3-4 | Visual calendar with drag-and-drop scheduling, week/month views, post status indicators |
| 5-6 | Local media library with thumbnail generation, bulk import, drag-to-attach workflow |
| 7 | Offline queue with auto-publish on reconnect, system tray with schedule status, native notifications |
| 8 | Auto-update, packaging (DMG/MSI/AppImage), onboarding flow, beta testing with 5-10 content creators |

**Post-MVP:** AI caption generation, analytics dashboard, team collaboration, external editor round-trip, template library, TikTok/Threads support
