# MeetingTLDR Desktop — 5-Framework Comparison

## Desktop Rationale

MeetingTLDR (AI meeting transcription and summarization) is one of the most compelling desktop use cases:

- **Local audio capture:** Capture both system audio (meeting app output) and microphone input simultaneously using OS-level audio APIs — impossible from a browser without screen sharing permissions and quality loss.
- **On-device transcription:** Run OpenAI Whisper locally for real-time speech-to-text without sending audio to the cloud. Eliminates API costs ($0.006/min adds up fast) and latency.
- **Privacy-first architecture:** Meeting audio never leaves the machine. Critical for confidential discussions — legal reviews, HR conversations, board meetings, M&A calls — where cloud transcription is a compliance risk.
- **Calendar app integration:** Read native calendar events (EventKit on macOS, Outlook COM on Windows) to auto-detect meeting start times, pre-fill meeting metadata, and correlate transcripts with calendar entries.
- **Real-time transcription overlay:** Display a floating, always-on-top transcription window during meetings — like live captions but with speaker identification and smart formatting.

---

## Desktop-Specific Features

- **System audio capture:** Capture audio output from Zoom, Teams, Google Meet, or any application via virtual audio driver or OS audio APIs
- **Microphone capture:** Simultaneous mic recording for speaker identification ("this is me vs. remote participants")
- **Real-time transcription overlay:** Floating, semi-transparent window showing live transcript during calls
- **Whisper local inference:** On-device speech-to-text using whisper.cpp (CPU) or whisper-metal (Apple Silicon GPU)
- **Calendar integration:** Auto-detect meetings from macOS Calendar / Outlook; pre-fill meeting title, attendees, and agenda
- **Auto-start recording:** Detect when a meeting app (Zoom, Teams, Meet) launches and prompt to start recording
- **Speaker diarization:** Identify and label different speakers in the transcript using on-device ML
- **Meeting summary generation:** AI-powered summary with action items, decisions, and key topics extracted from transcript
- **Searchable transcript archive:** Full-text search across all past meeting transcripts with timestamp navigation
- **Export formats:** Export transcripts and summaries to Notion, Google Docs, Confluence, Slack, or markdown files

---

## Shared Packages

### core (Rust)

- **Audio pipeline:** Mix system audio and microphone streams, normalize volume, apply noise reduction (via RNNoise), resample to 16kHz for Whisper
- **Transcription engine:** Whisper.cpp bindings for on-device speech-to-text with streaming (partial results as audio arrives)
- **Speaker diarization:** Voice activity detection + speaker embedding extraction for identifying who said what
- **Summarizer prompt builder:** Construct LLM prompts from transcripts — extract action items, decisions, topics, and follow-ups
- **Transcript formatter:** Convert raw ASR output into formatted transcript with timestamps, speaker labels, and paragraph breaks
- **Search indexer:** Full-text search across transcript content with timestamp-accurate navigation and speaker filtering

### api-client (Rust)

- **Cloud AI:** OpenAI/Anthropic API for cloud-based summarization when local inference is not available or for higher quality
- **Export integrations:** Push summaries to Notion (API), Google Docs (API), Confluence (REST), Slack (webhook)
- **Cloud sync:** Sync transcripts and summaries across devices for searchable meeting archive
- **Auth:** OAuth2 for export integrations; API key for cloud AI services; encrypted credential storage
- **Calendar API:** Google Calendar API and Microsoft Graph API fallback when native calendar access is unavailable

### data (SQLite)

- **Schema:** meetings (id, title, start_time, end_time, attendees[], calendar_event_id), transcripts (meeting_id, speaker, text, timestamp_start, timestamp_end), summaries, action_items, audio_files, settings
- **Local-first:** All transcripts and audio stored locally; cloud sync is optional and selective
- **FTS5 index:** Full-text search across transcript text with speaker and timestamp metadata
- **Audio file management:** Track audio file paths with duration, size, and encoding metadata; configurable auto-cleanup policies

---

## Framework Implementations

### Electron

**Architecture:** Main process handles audio capture (via native addon or spawned process), Whisper inference (via whisper-node or spawned whisper.cpp), and calendar access. Renderer (React) displays real-time transcript, summary editor, and meeting archive.

**Tech stack:**
- Renderer: React 18, TailwindCSS, Zustand, react-virtualized (transcript scrolling)
- Main: node-audiorecorder or native addon for system audio, whisper-node (Whisper bindings), better-sqlite3
- Database: better-sqlite3 with Drizzle ORM
- AI: whisper-node for transcription, OpenAI SDK for summarization

**Native module needs:** better-sqlite3, portaudio (native audio capture), whisper-node (whisper.cpp bindings)

**Bundle size:** ~250-350 MB (Chromium + Whisper model ~150 MB for base model)

**Memory:** ~400-700 MB (Whisper model loaded + Chromium + audio buffers)

**Pros:**
- React UI enables rich transcript viewer with speaker colors, timestamps, and clickable navigation
- Electron's always-on-top BrowserWindow works for the transcription overlay
- Mature ecosystem for text formatting, markdown export, and rich text editing
- Can reuse SaaS web frontend for the meeting archive and search views

**Cons:**
- Extremely heavy memory usage — Whisper model + Chromium is 500+ MB during active transcription
- Audio capture on the main process is unreliable — requires native addons that are fragile across OS updates
- System audio capture (loopback) is not natively supported by Electron — requires external virtual audio driver installation
- CPU contention between Chromium rendering and Whisper inference degrades transcription quality
- Battery drain during long meetings is significant due to Chromium's constant rendering overhead

### Tauri

**Architecture:** Rust backend handles audio capture (cpal crate), Whisper inference (whisper-rs), and all processing. Lightweight Svelte frontend for transcript display, summary editor, and meeting archive.

**Tech stack:**
- Frontend: Svelte 5, TailwindCSS
- Backend: cpal (audio capture), whisper-rs (whisper.cpp bindings), rusqlite, tokio
- Audio: cpal + platform-specific loopback (BlackHole on macOS, WASAPI loopback on Windows)
- AI: whisper-rs for transcription, reqwest to cloud API for summarization

**Plugin needs:** tauri-plugin-notification, tauri-plugin-autostart, tauri-plugin-global-shortcut, tauri-plugin-dialog

**Bundle size:** ~12-25 MB (+ Whisper model ~150 MB downloaded separately)

**Memory:** ~150-350 MB (Whisper model loaded; lower idle memory when not transcribing)

**Pros:**
- whisper-rs provides the most efficient Whisper inference on CPU — Rust bindings to whisper.cpp with zero overhead
- cpal gives low-level audio capture control with proper sample rate and format handling
- Low idle memory when not actively transcribing — suitable for always-running background app
- Tiny base bundle — Whisper model can be lazy-downloaded on first use
- Rust audio processing pipeline (denoising, mixing) runs efficiently without blocking the UI

**Cons:**
- System audio loopback requires external virtual audio driver on macOS (BlackHole, Soundflower) — adds onboarding friction
- No native calendar access from Rust — must use platform commands or calendar file parsing (ical)
- Real-time transcription overlay window requires careful Tauri window management and positioning
- Building a polished transcript viewer with live-updating auto-scroll in Svelte requires effort

### Flutter Desktop

**Architecture:** Custom transcript widget with auto-scrolling. Platform channels for audio capture and calendar access. Rust FFI for Whisper inference via flutter_rust_bridge.

**Tech stack:**
- UI: Flutter 3.x, Riverpod, custom transcript ListView, window_manager
- Database: drift
- Audio: platform channels → native audio capture APIs
- AI: flutter_rust_bridge → whisper-rs
- Calendar: platform channels → EventKit (macOS) / Outlook COM (Windows)

**Plugin needs:** window_manager, tray_manager, local_notifier, record (audio recording)

**Bundle size:** ~30-45 MB (+ Whisper model)

**Memory:** ~180-350 MB (during active transcription)

**Pros:**
- Custom transcript widget can provide smooth auto-scrolling with speaker avatars and animations
- Platform channels enable native calendar access on both macOS (EventKit) and Windows (Outlook)
- Could extend to a mobile companion app for reviewing meeting summaries on the go
- Hot reload is useful for iterating on the real-time transcript overlay layout

**Cons:**
- Audio capture via platform channels is complex and platform-specific — no unified API across macOS/Windows/Linux
- Whisper inference via FFI bridge adds latency to the transcription pipeline compared to native Rust calls
- Always-on-top overlay window is difficult to implement correctly in Flutter desktop
- Audio processing performance through the Dart/FFI boundary is suboptimal for real-time streaming use
- Battery consumption during multi-hour meetings is higher than native alternatives

### Swift/SwiftUI (macOS)

**Architecture:** SwiftUI views with ScreenCaptureKit for system audio capture. Speech framework or whisper.cpp via Swift package for transcription. EventKit for calendar. AVAudioEngine for microphone.

**Tech stack:**
- UI: SwiftUI, NSPanel (floating transcript overlay), native List for transcript viewer
- Audio: ScreenCaptureKit (system audio), AVAudioEngine (microphone), AudioToolbox
- AI: whisper.cpp via Swift bindings, or MLX-based Whisper model for Apple Silicon, CoreML
- Calendar: EventKit (native calendar access)
- Data: SwiftData

**Bundle size:** ~8-15 MB (+ Whisper CoreML model ~80 MB, more compact than ONNX)

**Memory:** ~60-150 MB (CoreML manages Whisper model memory efficiently via ANE)

**Pros:**
- **ScreenCaptureKit provides the best system audio capture API** — captures app audio without virtual audio drivers
- **MLX/CoreML Whisper runs on Apple Neural Engine** — 3-5x faster than CPU inference, lower power consumption
- EventKit gives direct, permission-based native calendar access with change notifications
- NSPanel provides the perfect floating overlay — non-activating, always-on-top, click-through capable
- AVAudioEngine + ScreenCaptureKit together provide the cleanest dual-stream (system + mic) audio capture
- Lowest power consumption during long meeting recordings — critical for laptop battery life
- Native speech recognition (Speech framework) available as fallback — no model download needed

**Cons:**
- macOS only — teams with Windows and Linux users cannot use it, cross-platform is important for team tools
- ScreenCaptureKit requires macOS 13+ for audio capture features (excludes older macOS versions)
- MLX Whisper models are Apple Silicon only — Intel Macs fall back to slower CPU inference
- Building the summary and export features (Notion, Confluence, Google Docs) is more work in Swift than in web frameworks
- App Store distribution requires explaining microphone and screen recording permissions clearly

### Kotlin Multiplatform

**Architecture:** Compose Multiplatform for UI. JNA for native audio capture. DJL or whisper-jni for Whisper inference. Ktor for API integrations.

**Tech stack:**
- UI: Compose Desktop, Material 3, custom transcript composable
- Database: SQLDelight
- Audio: JNA → PortAudio or platform-native audio APIs
- AI: whisper-jni (JNI bindings to whisper.cpp), or DJL with ONNX Runtime
- Calendar: JNA → EventKit (macOS), Outlook COM (Windows)
- HTTP: Ktor client

**Bundle size:** ~60-90 MB (+JRE + Whisper model)

**Memory:** ~300-500 MB (JVM + Whisper model + audio buffers)

**Pros:**
- Could share meeting archive and search logic with an Android companion app
- DJL provides flexible ML inference abstractions across multiple backends
- Ktor client handles OAuth2 for export integrations cleanly
- Kotlin coroutines map well to concurrent audio capture + transcription + UI update

**Cons:**
- Highest memory usage — JVM + Whisper model + audio processing exceeds 400 MB during transcription
- JNA audio capture is an indirect path to native APIs — adds latency and complexity to the audio pipeline
- JVM is not optimized for real-time audio processing — garbage collection pauses can cause audio glitches and dropped frames
- Compose Desktop floating window (for transcription overlay) is less polished than native implementations
- Whisper inference on JVM (via JNI) has overhead compared to direct C/Rust bindings
- Battery impact during long meetings is the worst of all options due to JVM overhead

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 8 | 9 | 11 | 7 | 11 |
| Bundle size | 300 MB | 20 MB | 38 MB | 12 MB | 78 MB |
| Memory usage | 550 MB | 250 MB | 280 MB | 100 MB | 420 MB |
| Startup time | 3.0s | 0.9s | 1.4s | 0.4s | 2.5s |
| Native feel | 4/10 | 6/10 | 5/10 | 10/10 | 4/10 |
| Offline capability | 7/10 | 9/10 | 7/10 | 9/10 | 7/10 |
| Audio capture quality | 5/10 | 7/10 | 6/10 | 10/10 | 4/10 |
| Cross-platform | 9/10 | 9/10 | 9/10 | 2/10 | 8/10 |
| Best fit score | 5/10 | 8/10 | 5/10 | 9/10 | 4/10 |

---

## Recommended Framework

**Swift/SwiftUI** is the best fit for MeetingTLDR Desktop on macOS. **Tauri** is recommended for cross-platform.

**Rationale:** MeetingTLDR's core value depends entirely on audio capture quality and transcription performance — the two areas where Swift on macOS is unmatched. ScreenCaptureKit provides system audio capture without virtual audio drivers (a massive UX improvement), and MLX Whisper on Apple Neural Engine delivers 3-5x faster transcription with lower power consumption. For a meeting tool used during hours-long calls, battery efficiency and audio reliability are non-negotiable. If cross-platform is required, Tauri with whisper-rs provides the best balance of transcription performance and lightweight resource usage, though system audio capture will require virtual audio drivers on macOS.

**Runner-up:** Tauri for cross-platform deployments where Windows and Linux support is needed. The Rust audio and transcription pipeline (cpal + whisper-rs) is solid, though the macOS audio capture experience will be inferior to the Swift version.

---

## Monetization (Desktop)

- **Free tier:** Local transcription only (Whisper base model), 5 meetings/month, no cloud summarization, basic text export
- **Pro ($19/mo or $159/yr):** Unlimited meetings, AI summaries (cloud), action item extraction, Notion/Google Docs export, larger Whisper model download
- **Team ($12/user/mo):** Shared meeting archive, team search, collaborative meeting notes, Slack integration, speaker analytics
- **Enterprise ($24/user/mo):** SSO, compliance-grade local-only mode (no cloud), custom vocabulary, API access, admin controls
- **Distribution:** Direct download; Mac App Store (for the Swift version); Homebrew cask for developers

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | Audio capture pipeline (system audio + microphone), recording to local WAV/M4A files |
| 3-4 | Whisper local transcription (base model), real-time transcript display with timestamps |
| 5-6 | AI summarization (cloud API), meeting archive with full-text search, transcript editor |
| 7 | Calendar integration (auto-detect meetings), floating transcription overlay, system tray controls |
| 8 | Auto-update, packaging (DMG for macOS, MSI/AppImage for Tauri), export to Markdown/Notion, beta testing |

**Post-MVP:** Speaker diarization, larger Whisper models (medium/large), team meeting archive, Confluence/Slack export, recording trimmer, custom vocabulary, real-time translation
