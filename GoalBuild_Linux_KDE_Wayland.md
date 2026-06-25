# Goal: Build a Linux KDE Wayland Granola-like Local/Cloud Call Transcription and Summarization App

You are GPT-5.5 running as an agentic coding assistant. Your goal is to iteratively implement a working MVP desktop application for Linux, specifically targeting **Fedora KDE Plasma on Wayland**.

The app should be a **Granola-like meeting transcription and summarization tool** that captures audio locally from the device, not by joining the call as a bot. It must work with online calls in browsers and desktop apps, especially:

* Google Meet in Chrome/Chromium/Brave
* Zoom web
* Microsoft Teams web
* Slack huddles/web calls where possible
* Discord/browser calls where possible

The MVP must be Rust-based, Tauri-based, and must capture both microphone audio and the audio coming from the call/browser/system output on Fedora KDE Wayland using PipeWire.

The final MVP must allow the user to:

1. Start a meeting recording from a Tauri desktop UI.
2. Capture microphone audio.
3. Capture system/browser/call audio from what the user hears on KDE Wayland.
4. Preserve mic and system audio as separate streams where possible.
5. Transcribe the call.
6. Attribute speech at least as `You` vs `Others`.
7. Optionally perform stronger speaker diarization when a provider supports it.
8. Summarize the meeting.
9. Produce structured meeting notes, action items, decisions, and follow-ups.
10. Use local models where possible.
11. Use cloud/self-hosted/OpenAI-compatible endpoints where configured.
12. Never require a subscription or proprietary hosted backend from this app itself.
13. Store all user meeting data locally by default.
14. Clearly expose privacy modes: local-only, cloud STT, cloud LLM, cloud STT + cloud LLM.

Do not merely scaffold a UI. Build a working end-to-end product. However, implement it iteratively and keep the codebase modular, testable, and debuggable.

---

## Non-negotiable MVP target

The MVP is successful when, on Fedora KDE Plasma Wayland, the user can:

1. Open Google Meet in a Chromium-based browser.
2. Join a call.
3. Open this app.
4. Click `Start Recording`.
5. Speak locally and hear remote participants.
6. Click `Stop Recording`.
7. Click `Transcribe`.
8. Click `Summarize`.
9. See a useful transcript and summary inside the Tauri app.
10. Export a Markdown file containing transcript, summary, decisions, and action items.

The app must not use a meeting bot. It must capture from the local device audio stack.

---

## Platform target

Primary target:

```text
OS: Fedora Linux
Desktop: KDE Plasma
Session: Wayland
Audio stack: PipeWire
App shell: Tauri v2
Backend: Rust
Frontend: Svelte, React, or another simple Tauri-compatible web frontend
```

Secondary Linux support may be added later, but do not optimize for GNOME first. KDE Wayland on Fedora is the priority.

---

## Technical strategy

Implement a Rust workspace with a Tauri app plus reusable backend crates.

The Tauri UI should be thin. Core functionality should live in Rust crates that can also be tested by CLI commands.

The core architecture must use provider abstractions:

```rust
AudioBackend
TranscriptionProvider
DiarizationProvider
SummaryProvider
MeetingRepository
ExportProvider
```

Do not hardcode one model provider. The app must support both local and cloud/self-hosted providers.

---

## Repository structure

Create or migrate to this structure:

```text
meeting-notes/
  Cargo.toml
  AGENTS.md
  README.md

  docs/
    architecture.md
    fedora-kde-wayland-test-plan.md
    privacy-model.md
    provider-api.md
    troubleshooting-audio.md

  crates/
    audio-core/
    audio-pipewire/
    audio-cpal-fallback/
    dsp/
    vad/
    recorder/
    asr/
    diarization/
    llm/
    meeting-core/
    storage/
    config/
    diagnostics/
    export/

  apps/
    cli/
    daemon/
    tauri-app/
```

The app may include CLI binaries for internal testing, but the final MVP must have a Tauri UI.

---

## Development rules

Follow these rules throughout the project:

1. Do not put PipeWire capture logic directly inside UI components.
2. Do not put transcription logic directly inside Tauri commands.
3. Do not put summarization prompt logic directly inside frontend code.
4. Do not mix microphone and system audio too early.
5. Preserve separate `mic` and `system` tracks where possible.
6. Always write diagnostics for each recording session.
7. Make every major subsystem testable outside the UI.
8. Prefer sidecar process integration before native FFI for complex ML tools.
9. Do not silently send audio or transcript to cloud APIs.
10. Every cloud provider must require explicit config.
11. Use local-only mode as the default.
12. When uncertain about Linux audio behavior, build a diagnostic command or UI diagnostic panel rather than guessing.
13. Prefer boring, robust architecture over clever abstractions.

---

## Workspace dependencies

Use stable and well-known Rust dependencies. Add new dependencies only when necessary.

Suggested baseline dependencies:

```toml
anyhow = "1"
thiserror = "2"
tracing = "0.1"
tracing-subscriber = "0.3"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["full"] }
clap = { version = "4", features = ["derive"] }
hound = "3"
rubato = "0.16"
reqwest = { version = "0.12", features = ["json", "multipart", "stream"] }
sqlx = { version = "0.8", features = ["sqlite", "runtime-tokio-rustls"] }
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
directories = "6"
```

Use PipeWire Rust bindings for the PipeWire backend. If direct PipeWire Rust integration becomes blocked, use a controlled fallback via GStreamer or command-sidecar integration, but keep the `AudioBackend` trait stable.

---

## Core data model

Implement these core types in `crates/meeting-core`.

```rust
pub enum AudioSourceKind {
    Microphone,
    SystemOutputMonitor,
    Application,
    Unknown,
}

pub struct AudioSource {
    pub id: String,
    pub name: String,
    pub description: Option<String>,
    pub kind: AudioSourceKind,
    pub is_default: bool,
}

pub enum AudioTrackKind {
    Mic,
    System,
    Mixed,
}

pub struct Meeting {
    pub id: String,
    pub title: String,
    pub started_at: DateTime<Utc>,
    pub ended_at: Option<DateTime<Utc>>,
    pub status: MeetingStatus,
    pub root_dir: PathBuf,
}

pub enum MeetingStatus {
    Idle,
    Starting,
    Recording,
    Stopping,
    Finalizing,
    ReadyForTranscription,
    Transcribing,
    ReadyForSummary,
    Summarizing,
    Complete,
    Failed,
}

pub struct TranscriptSegment {
    pub id: String,
    pub meeting_id: String,
    pub source: AudioTrackKind,
    pub speaker: Option<String>,
    pub start_ms: u64,
    pub end_ms: u64,
    pub text: String,
    pub confidence: Option<f32>,
}

pub struct MeetingSummary {
    pub overview: String,
    pub decisions: Vec<Decision>,
    pub action_items: Vec<ActionItem>,
    pub open_questions: Vec<OpenQuestion>,
    pub follow_up_draft: Option<String>,
}

pub struct ActionItem {
    pub task: String,
    pub owner: Option<String>,
    pub due: Option<String>,
    pub evidence_segment_ids: Vec<String>,
}
```

Keep schema serialization stable and versioned.

---

## Session directory format

Every meeting session must produce a local folder.

Example:

```text
~/.local/share/meeting-notes/meetings/<meeting-id>/
  meeting.json
  capture.json
  events.jsonl

  audio/
    mic.wav
    system.wav
    mixed-preview.wav

  chunks/
    mic-000001.wav
    system-000001.wav
    ...

  chunks.json
  transcript.json
  transcript.md
  diarization.json
  summary.json
  summary.md
  export.md
```

`meeting.json` stores high-level meeting metadata.

`capture.json` stores audio device and capture details.

`events.jsonl` stores structured logs for debugging.

---

## Diagnostics are mandatory

Every recording session must produce diagnostics.

Include at least:

```json
{
  "app_version": "0.1.0",
  "os": "linux",
  "desktop": "kde",
  "session_type": "wayland",
  "pipewire_detected": true,
  "pipewire_version": null,
  "selected_sources": {
    "mic": {
      "id": "...",
      "name": "...",
      "sample_rate": 48000,
      "channels": 2
    },
    "system": {
      "id": "...",
      "name": "...monitor",
      "sample_rate": 48000,
      "channels": 2
    }
  },
  "recording": {
    "duration_ms": 0,
    "mic_frames": 0,
    "system_frames": 0,
    "dropped_frames": 0,
    "underruns": 0,
    "overruns": 0
  }
}
```

Also log events like:

```jsonl
{"ts":"...","level":"info","event":"session.starting"}
{"ts":"...","level":"info","event":"pipewire.connected"}
{"ts":"...","level":"info","event":"mic.stream.ready"}
{"ts":"...","level":"info","event":"system.stream.ready"}
{"ts":"...","level":"warn","event":"buffer.underrun","source":"system"}
{"ts":"...","level":"info","event":"session.finalized"}
```

The UI must include a diagnostics/export-debug-bundle button.

---

## Audio architecture

Implement these crates:

```text
crates/audio-core
crates/audio-pipewire
crates/audio-cpal-fallback
crates/dsp
crates/recorder
```

### AudioBackend trait

In `audio-core`, define:

```rust
#[async_trait]
pub trait AudioBackend: Send + Sync {
    async fn list_sources(&self) -> anyhow::Result<Vec<AudioSource>>;
    async fn get_default_microphone(&self) -> anyhow::Result<Option<AudioSource>>;
    async fn get_default_system_monitor(&self) -> anyhow::Result<Option<AudioSource>>;
    async fn start_capture(&self, config: CaptureConfig) -> anyhow::Result<Box<dyn CaptureSession>>;
}

pub trait CaptureSession: Send {
    fn stop(&mut self) -> anyhow::Result<CaptureResult>;
}
```

### Linux implementation

Implement `PipeWireAudioBackend`.

It must:

1. Connect to PipeWire.
2. Enumerate available audio nodes.
3. Detect microphones.
4. Detect output monitor sources.
5. Select default microphone.
6. Select default system output monitor.
7. Capture microphone audio.
8. Capture system output audio.
9. Record both streams concurrently.
10. Recover gracefully where possible if a device changes.
11. Produce useful errors if capture fails.

Do not initially require per-application capture. For MVP, capturing the default output monitor is enough if it captures browser meeting audio.

Later, add optional app-specific stream selection if feasible.

### Required KDE Wayland behavior

The app must work under KDE Plasma Wayland with PipeWire. It should not depend on X11 APIs.

The app should detect:

```text
XDG_CURRENT_DESKTOP
XDG_SESSION_TYPE
KDE_FULL_SESSION
PIPEWIRE_RUNTIME_DIR
```

If the system is not Wayland/PipeWire, the app should show a useful warning.

---

## Audio processing

In `crates/dsp`, implement:

1. Resampling to 16 kHz.
2. Mono conversion.
3. WAV writing.
4. Optional gain normalization.
5. Optional silence trimming.
6. Chunking into transcription-ready files.

Recommended internal format:

```text
f32 PCM internally
16 kHz mono for ASR chunks
WAV output for debugging and sidecar compatibility
```

Chunking rules:

```text
target chunk length: 20-30 seconds
max chunk length: 45 seconds
min chunk length: 3 seconds
preserve absolute timestamps
preserve source track: mic or system
```

---

## VAD

Implement a basic VAD first and make it replaceable.

Provider interface:

```rust
pub trait VadProvider {
    fn detect_speech(&self, audio: &[f32], sample_rate: u32) -> anyhow::Result<Vec<SpeechSpan>>;
}
```

Initial implementation:

```text
EnergyThresholdVad
```

Optional later implementations:

```text
SileroVadProvider
WebRtcVadProvider
SherpaOnnxVadProvider
```

The chunker must work even if VAD is disabled.

---

## Recording flow

Implement `MeetingRecorder`.

Expected API:

```rust
impl MeetingRecorder {
    pub async fn start(config: RecordingConfig) -> anyhow::Result<MeetingHandle>;
    pub async fn stop(meeting_id: &str) -> anyhow::Result<CaptureResult>;
}
```

Recording must create:

```text
audio/mic.wav
audio/system.wav
audio/mixed-preview.wav
capture.json
events.jsonl
```

If the system audio source is unavailable, the app may allow mic-only recording but must show a warning that call audio will not be captured.

---

## Transcription architecture

Implement `crates/asr`.

Define:

```rust
#[async_trait]
pub trait TranscriptionProvider: Send + Sync {
    fn id(&self) -> &'static str;
    fn mode(&self) -> ProviderMode;
    fn supports_streaming(&self) -> bool;
    fn supports_word_timestamps(&self) -> bool;
    fn supports_speaker_diarization(&self) -> bool;

    async fn transcribe_file(
        &self,
        input: TranscriptionInput
    ) -> anyhow::Result<TranscriptionOutput>;
}
```

Where:

```rust
pub enum ProviderMode {
    Local,
    LocalHttp,
    Cloud,
    SelfHosted,
}

pub struct TranscriptionInput {
    pub audio_path: PathBuf,
    pub source: AudioTrackKind,
    pub speaker_hint: Option<String>,
    pub start_offset_ms: u64,
    pub language: Option<String>,
}

pub struct TranscriptionOutput {
    pub segments: Vec<TranscriptSegment>,
    pub raw_provider_response: Option<serde_json::Value>,
}
```

### Required STT providers for MVP or near-MVP

Implement these provider adapters:

#### 1. Whisper.cpp local sidecar

Use local `whisper.cpp` binary or server.

Support config:

```toml
[asr.whisper_cpp]
enabled = true
mode = "sidecar"
binary_path = "/usr/local/bin/whisper-cli"
server_url = "http://127.0.0.1:2022/v1"
model_path = "~/.local/share/meeting-notes/models/ggml-small.bin"
language = "auto"
```

It should support either:

* spawning a local binary; or
* calling a local OpenAI-compatible whisper.cpp server.

#### 2. OpenAI-compatible STT endpoint

Support:

```toml
[asr.openai_compatible]
enabled = true
base_url = "https://..."
api_key_env = "OPENAI_API_KEY"
model = "whisper-1"
```

Use this for OpenAI-like endpoints, self-hosted endpoints, or compatible cloud vendors.

#### 3. Azure Speech / Microsoft diarization-capable provider

Implement a provider abstraction that can call Microsoft/Azure Speech STT with diarization when configured.

Support config:

```toml
[asr.azure_speech]
enabled = false
endpoint = "https://..."
region = "..."
api_key_env = "AZURE_SPEECH_KEY"
language = "en-US"
enable_diarization = true
```

If real-time Azure diarization is not implemented immediately, implement the config, provider trait, clear TODO, and a batch/file-based provider path if available. The app architecture must not block adding it.

#### 4. Future providers

Design the provider registry so these can be added later:

```text
faster-whisper
sherpa-onnx
Deepgram
AssemblyAI
Groq STT
Fireworks/Whisper endpoints
local Vosk
local ONNX models
```

---

## Diarization architecture

Diarization must support two levels:

### Level 1: channel/source-based attribution

This is required for MVP.

Use:

```text
mic track => speaker = "You"
system track => speaker = "Others"
```

This is not true diarization, but it is extremely useful and reliable for local meeting capture.

### Level 2: provider diarization

Implement `crates/diarization`.

Define:

```rust
#[async_trait]
pub trait DiarizationProvider: Send + Sync {
    fn id(&self) -> &'static str;
    fn mode(&self) -> ProviderMode;

    async fn diarize(
        &self,
        input: DiarizationInput
    ) -> anyhow::Result<DiarizationOutput>;
}
```

Schema:

```rust
pub struct SpeakerTurn {
    pub speaker_id: String,
    pub start_ms: u64,
    pub end_ms: u64,
    pub confidence: Option<f32>,
}
```

Supported provider types:

```text
SourceBasedDiarizationProvider
AzureSpeechDiarizationProvider
PyannoteSidecarProvider
SherpaOnnxDiarizationProvider
```

MVP must support `SourceBasedDiarizationProvider`.

The architecture must allow Azure/Microsoft diarization-capable transcription and external diarization providers to return speaker labels.

The UI must allow renaming speakers:

```text
You
Speaker 1
Speaker 2
Speaker 3
```

Store speaker rename data locally per meeting.

---

## Transcription merge logic

After transcribing mic and system chunks separately, merge them by timestamps.

Algorithm:

1. Transcribe mic chunks with `speaker_hint = "You"`.
2. Transcribe system chunks with `speaker_hint = "Others"` or provider diarization labels.
3. Normalize all segment timestamps to absolute meeting time.
4. Sort all segments by `start_ms`.
5. Merge adjacent segments from same speaker if close enough.
6. Produce `transcript.json`.
7. Produce readable `transcript.md`.

Readable transcript format:

```md
# Transcript

[00:00:03] You:
Let's start with the Linux capture issue.

[00:00:07] Speaker 1:
I think KDE Wayland is the main risk.

[00:00:12] You:
Right, we need PipeWire monitor capture.
```

---

## Summarization architecture

Implement `crates/llm`.

Define:

```rust
#[async_trait]
pub trait SummaryProvider: Send + Sync {
    fn id(&self) -> &'static str;
    fn mode(&self) -> ProviderMode;

    async fn summarize(
        &self,
        input: SummaryInput
    ) -> anyhow::Result<MeetingSummary>;
}
```

Input:

```rust
pub struct SummaryInput {
    pub meeting_title: String,
    pub transcript_segments: Vec<TranscriptSegment>,
    pub user_notes: Option<String>,
    pub template: SummaryTemplate,
}
```

Required providers:

### 1. Ollama provider

Support:

```toml
[llm.ollama]
enabled = true
base_url = "http://localhost:11434"
model = "qwen2.5:7b"
```

### 2. OpenAI-compatible chat provider

Support:

```toml
[llm.openai_compatible]
enabled = true
base_url = "https://..."
api_key_env = "OPENAI_API_KEY"
model = "..."
```

This should work with:

```text
OpenAI
OpenRouter
Together
Groq
Fireworks
vLLM
llama.cpp server if compatible
LM Studio
self-hosted compatible endpoints
```

### 3. Local llama.cpp or LM Studio endpoints

If they expose OpenAI-compatible endpoints, use the same provider. Otherwise add a specific provider adapter.

---

## Summary output

The app must generate:

```text
Overview
Key points
Decisions
Action items
Open questions
Risks/blockers
Follow-up draft
```

Structured JSON:

```json
{
  "overview": "...",
  "key_points": [
    {
      "text": "...",
      "evidence_segment_ids": ["seg-000001"]
    }
  ],
  "decisions": [
    {
      "text": "...",
      "evidence_segment_ids": ["seg-000002"]
    }
  ],
  "action_items": [
    {
      "task": "...",
      "owner": null,
      "due": null,
      "evidence_segment_ids": ["seg-000003"]
    }
  ],
  "open_questions": [],
  "risks": [],
  "follow_up_draft": "..."
}
```

The summarizer must be instructed not to invent facts. It should preserve uncertainty and use evidence from transcript segments.

---

## Prompting requirements for summarization

Use a prompt similar to this:

```text
You are a meeting note assistant.

You will receive:
- Meeting title
- Optional user notes
- Timestamped transcript segments with speaker labels

Your task:
- Create concise, useful meeting notes.
- Do not invent facts not supported by transcript or user notes.
- Preserve uncertainty.
- Extract decisions only if they were actually made.
- Extract action items only if they were assigned or strongly implied.
- Use speaker names if available.
- Include evidence segment IDs for key claims.

Return valid JSON matching the requested schema.
```

If the provider fails to return valid JSON, retry once with a stricter repair prompt. If it still fails, save the raw output and show a useful error.

---

## Storage architecture

Implement `crates/storage` with SQLite.

Minimum tables:

```sql
CREATE TABLE meetings (
  id TEXT PRIMARY KEY,
  title TEXT NOT NULL,
  started_at TEXT NOT NULL,
  ended_at TEXT,
  status TEXT NOT NULL,
  root_dir TEXT NOT NULL,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE TABLE transcript_segments (
  id TEXT PRIMARY KEY,
  meeting_id TEXT NOT NULL,
  source TEXT NOT NULL,
  speaker TEXT,
  start_ms INTEGER NOT NULL,
  end_ms INTEGER NOT NULL,
  text TEXT NOT NULL,
  confidence REAL,
  raw_json TEXT,
  FOREIGN KEY(meeting_id) REFERENCES meetings(id)
);

CREATE TABLE summaries (
  meeting_id TEXT PRIMARY KEY,
  json TEXT NOT NULL,
  markdown TEXT NOT NULL,
  provider_id TEXT NOT NULL,
  model TEXT,
  created_at TEXT NOT NULL,
  FOREIGN KEY(meeting_id) REFERENCES meetings(id)
);

CREATE TABLE speaker_aliases (
  id TEXT PRIMARY KEY,
  meeting_id TEXT NOT NULL,
  original_speaker_id TEXT NOT NULL,
  display_name TEXT NOT NULL,
  FOREIGN KEY(meeting_id) REFERENCES meetings(id)
);
```

Large audio files and chunk files should stay on disk, referenced by SQLite.

---

## Config architecture

Use:

```text
~/.config/meeting-notes/config.toml
```

Example:

```toml
[audio]
backend = "pipewire"
mic_source = "default"
system_source = "default-monitor"
sample_rate = 16000
channels = 1
record_mixed_preview = true

[recording]
save_audio = "until_transcribed"
chunk_seconds = 30
vad = "energy"

[asr]
provider = "whisper_cpp"

[asr.whisper_cpp]
mode = "server"
server_url = "http://127.0.0.1:2022/v1"
binary_path = ""
model_path = "~/.local/share/meeting-notes/models/ggml-small.bin"
language = "auto"

[asr.openai_compatible]
base_url = ""
api_key_env = "OPENAI_API_KEY"
model = "whisper-1"

[asr.azure_speech]
enabled = false
endpoint = ""
region = ""
api_key_env = "AZURE_SPEECH_KEY"
language = "en-US"
enable_diarization = true

[diarization]
provider = "source_based"

[llm]
provider = "ollama"

[llm.ollama]
base_url = "http://localhost:11434"
model = "qwen2.5:7b"

[llm.openai_compatible]
base_url = ""
api_key_env = "OPENAI_API_KEY"
model = ""

[privacy]
allow_cloud_asr = false
allow_cloud_llm = false
delete_audio_after_transcription = false
```

The UI must allow editing key provider settings.

---

## Tauri UI requirements

Implement a Tauri v2 app.

Frontend may be Svelte, React, or another lightweight framework. Choose one and keep it simple.

Required screens:

### 1. Home / Meeting list

Shows previous meetings:

```text
Title
Date
Duration
Status
Transcript available?
Summary available?
```

Actions:

```text
New Meeting
Open Meeting
Export
Delete
```

### 2. New Meeting / Recording screen

Contains:

```text
Meeting title input
Selected microphone
Selected system audio source
Audio level meter for mic
Audio level meter for system audio
Start Recording button
```

Before recording, the UI must verify that both mic and system sources are detectable. If system audio is missing, show a warning.

### 3. Active Recording screen

Shows:

```text
Recording duration
Mic level
System audio level
Status logs
Stop Recording button
```

### 4. Post-recording screen

Actions:

```text
Transcribe
Summarize
Export Markdown
Open diagnostics
```

### 5. Transcript screen

Shows timestamped transcript.

Features:

```text
speaker labels
rename speaker
search transcript
copy transcript
```

### 6. Summary screen

Shows:

```text
Overview
Key points
Decisions
Action items
Open questions
Follow-up draft
```

### 7. Settings screen

Provider settings:

```text
Audio backend
Mic source
System source
STT provider
STT model
Diarization provider
LLM provider
LLM model
Privacy mode
```

### 8. Diagnostics screen

Shows:

```text
PipeWire detection
KDE/Wayland detection
selected devices
capture health
dropped frames
underruns
debug bundle export
```

---

## Tauri backend API

Expose Tauri commands that call Rust backend services.

Commands:

```rust
#[tauri::command]
async fn list_audio_sources() -> Result<Vec<AudioSource>, String>;

#[tauri::command]
async fn create_meeting(title: String) -> Result<Meeting, String>;

#[tauri::command]
async fn start_recording(meeting_id: String) -> Result<(), String>;

#[tauri::command]
async fn stop_recording(meeting_id: String) -> Result<CaptureResult, String>;

#[tauri::command]
async fn transcribe_meeting(meeting_id: String) -> Result<Transcript, String>;

#[tauri::command]
async fn summarize_meeting(meeting_id: String) -> Result<MeetingSummary, String>;

#[tauri::command]
async fn list_meetings() -> Result<Vec<Meeting>, String>;

#[tauri::command]
async fn get_meeting(meeting_id: String) -> Result<MeetingDetail, String>;

#[tauri::command]
async fn export_markdown(meeting_id: String) -> Result<PathBuf, String>;

#[tauri::command]
async fn get_diagnostics(meeting_id: String) -> Result<Diagnostics, String>;

#[tauri::command]
async fn update_config(config: AppConfig) -> Result<(), String>;
```

Long-running operations should emit progress events to the frontend:

```text
recording.started
recording.levels
recording.stopped
transcription.started
transcription.progress
transcription.completed
summary.started
summary.completed
error
```

---

## CLI requirements

Even though the final product is Tauri, keep a CLI for testing and debugging.

Implement:

```bash
meeting-cli list-devices
meeting-cli record --title "Test" --seconds 60
meeting-cli transcribe <meeting-id>
meeting-cli summarize <meeting-id>
meeting-cli export <meeting-id>
meeting-cli diagnostics <meeting-id>
```

The CLI is allowed to share the same backend crates as the Tauri app.

---

## Manual test requirements on Fedora KDE Wayland

Create `docs/fedora-kde-wayland-test-plan.md`.

The MVP must be manually tested with:

```text
Fedora KDE Plasma Wayland
PipeWire active
Chromium or Chrome
Google Meet call
Built-in microphone
Built-in speakers
Headphones
Bluetooth headphones if available
```

Test cases:

### Test 1: Device detection

Expected:

```text
App detects Wayland session
App detects KDE
App detects PipeWire
App lists microphone
App lists system output monitor
```

### Test 2: Mic capture

Expected:

```text
mic.wav contains local user voice
system.wav may be silent
```

### Test 3: System/browser audio capture

Expected:

```text
Play YouTube in browser
system.wav contains browser audio
mic.wav does not contain browser audio except acoustic leakage
```

### Test 4: Google Meet capture

Expected:

```text
mic.wav contains local user voice
system.wav contains remote participant audio
transcript contains both sides
speaker labels separate You vs Others
summary reflects actual conversation
```

### Test 5: Headphones

Expected:

```text
system.wav still captures remote audio even when headphones are used
```

### Test 6: Provider switching

Expected:

```text
Ollama summary works locally
OpenAI-compatible LLM endpoint works when configured
Whisper.cpp or local STT works when configured
OpenAI-compatible STT endpoint works when configured
```

### Test 7: Privacy mode

Expected:

```text
In local-only mode, no cloud endpoint is called
If cloud provider is selected, UI clearly warns what data is sent
```

---

## Error handling requirements

The app must show useful errors for:

```text
PipeWire unavailable
No microphone found
No system monitor found
Permission or access failure
STT provider unavailable
LLM provider unavailable
Model missing
Ollama not running
Cloud API key missing
Cloud API request failed
Invalid JSON from LLM
Transcription failed for one chunk
```

Errors must be visible in UI and saved in diagnostics.

---

## Privacy requirements

Default mode:

```text
Local-only
No cloud STT
No cloud LLM
Audio stored locally
```

When cloud is enabled, show explicit labels:

```text
Cloud STT enabled: audio chunks may be sent to configured STT provider.
Cloud LLM enabled: transcript text may be sent to configured LLM provider.
```

Do not send data to any endpoint unless the selected provider requires it and the corresponding privacy flag is enabled.

---

## Markdown export format

Implement Markdown export:

```md
# Meeting Title

Date:
Duration:

## Summary

...

## Decisions

- ...

## Action Items

- [ ] Task — Owner — Due date

## Open Questions

- ...

## Transcript

[00:00:03] You:
...

[00:00:07] Speaker 1:
...
```

---

## Implementation behavior

Work iteratively. After each meaningful change:

1. Run `cargo fmt`.
2. Run `cargo check`.
3. Run tests if present.
4. If UI changed, run frontend typecheck/build.
5. Update docs if architecture or config changed.
6. Keep a short changelog in `docs/progress.md`.

If blocked by PipeWire API complexity, do not abandon the architecture. Instead:

1. Create a minimal reproducible diagnostic binary.
2. Log what works and what fails.
3. Try the simplest capture path first: default mic + default output monitor.
4. Only then consider GStreamer/Pulse compatibility fallback.

---

## Acceptance criteria

The MVP is complete when all are true:

```text
- Tauri app launches on Fedora KDE Wayland.
- App detects KDE Wayland and PipeWire.
- App lists microphone and system monitor sources.
- App can record mic audio.
- App can record browser/system audio.
- App can record Google Meet local + remote audio into separate tracks.
- App can transcribe recorded audio.
- App can label local user as You and remote/system audio as Others.
- App can optionally use diarization-capable providers when configured.
- App can summarize transcript using Ollama.
- App can summarize transcript using OpenAI-compatible endpoint when configured.
- App can use OpenAI-compatible STT endpoint when configured.
- App stores meetings locally.
- App exports Markdown notes.
- App produces diagnostics/debug bundle.
- App does not require a meeting bot.
- App does not require a subscription.
```

---

## Start now

Begin by creating the Rust workspace, Tauri app shell, core crate skeletons, config model, and audio provider interfaces.

Then implement Fedora KDE Wayland PipeWire device detection and diagnostics.

Then implement actual mic/system capture.

Continue iteratively until the MVP acceptance criteria pass.
