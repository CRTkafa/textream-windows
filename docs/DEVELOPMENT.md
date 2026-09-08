# Textream for Windows — Development Notes

This document contains implementation details, design rationale, and project scope that do not need to live in the main README.

## Architecture

```text
crates/prompt-core/     Platform-neutral engine — no OS calls, no audio device
├── text.rs             CJK-aware tokenisation, RTL direction inference
├── alignment.rs        [cue] handling and the commit policy
├── script.rs           Parsed script: words, ranges, progress conversions
├── matcher.rs          Character- + word-level speech alignment
├── vad.rs              Level metering and the hysteretic speech gate
└── scroll.rs           Constant-pace scrolling for Classic / Voice-Activated

src-tauri/              Windows shell
├── window_effects.rs   NOACTIVATE, click-through, exclude-from-capture
├── overlay.rs          Placement and multi-monitor geometry
├── audio.rs            Microphone capture and the speech worker
├── speech.rs           Streaming recogniser over the sherpa-onnx C API
├── model.rs            Model registry and first-run download
├── document.rs         .textream file format — read/write, macOS-compatible
├── backdrop.rs         Mica or blur behind the frameless window
├── settings.rs         Persisted preferences
├── shortcuts.rs        Global shortcuts for hands-free control
├── diagnostics.rs      Crash and background-error logging
├── session.rs          Live session state and the webview-facing DTOs
└── lib.rs              Commands, events, tray icon

src/                    Svelte UI
├── App.svelte          Editor, settings panel and transport dock
├── Chrome.svelte       Title bar and resize grips for the frameless window
├── LiquidStart.svelte  The voice-reactive transport control
├── Overlay.svelte      The prompter pill
└── lib/                Command wrappers, spring integrator, file dialogs, toasts
```

The engine deliberately knows nothing about Windows. That keeps it testable without a microphone and without launching the app.

## Design decisions

### Overlay placement

The prompter sits at the top of the screen instead of on the taskbar. The point is to keep the presenter looking close to the webcam rather than down toward the bottom of the monitor.

The taskbar strip is reserved for transport controls, while the app itself lives in the system tray.

### Privacy

Speech recognition is fully local. There is no account, telemetry, or cloud speech API.

### Word tracking

Word tracking is treated as an alignment problem rather than a general transcription problem. The script is already known, so the recogniser only needs to provide a sufficiently good transcript for the matcher to determine how far into the script the presenter has progressed.

## Windows specifics

| Concern | Mechanism |
|---|---|
| Never steals focus | `WS_EX_NOACTIVATE` |
| Out of Alt-Tab | `WS_EX_TOOLWINDOW` |
| Click-through overlay | `WS_EX_TRANSPARENT` |
| Hidden from capture | `SetWindowDisplayAffinity(WDA_EXCLUDEFROMCAPTURE)` |
| Always above the taskbar | Always-on-top window, not a registered AppBar |
| Single instance | `tauri-plugin-single-instance` |
| Close behavior | Main editor hides to tray instead of terminating the session |
| Quit during a take | Tray Quit asks for confirmation while a session is active |

## Guidance modes

| Mode | Driven by | Microphone |
|---|---|---|
| **Word Tracking** | `PromptMatcher` fed transcript windows | Required |
| **Classic** | `PaceScroller`, gate held open | Not needed |
| **Voice-Activated** | `PaceScroller` gated by `VoiceActivityDetector` | Required |

## Speech recognition

Word Tracking uses a streaming Zipformer transducer through sherpa-onnx.

Models are downloaded on first use into the app data directory and are not bundled into the installer.

`sherpa-rs` wraps sherpa-onnx's offline recogniser, while Textream needs continuous streaming recognition. For that reason, [`speech.rs`](../src-tauri/src/speech.rs) calls the sherpa-onnx online C API directly.

Audio capture also lives in Rust rather than in the webview. One capture path feeds both the level meter and the recogniser.

### Available models

| Language/model | Approx. size |
|---|---:|
| English (small) | 42 MB |
| English | 71 MB |
| German | 71 MB |
| French | 71 MB |
| Spanish | 156 MB |
| Mandarin Chinese | 77 MB |
| Arabic / Indonesian / Japanese / Russian / Thai / Vietnamese | 339 MB |

Turkish is not available yet because there is no compatible streaming sherpa-onnx model in the layout Textream expects.

Adding a language is mostly a model-registry change in [`model.rs`](../src-tauri/src/model.rs), provided a compatible published model exists.

## `.textream` files

`.textream` files use the same format as the macOS app: a JSON array of page strings.

The Windows port currently edits one continuous script. Multi-page files still open completely, but page boundaries are flattened into blank-line separators.

The current script is autosaved in app settings, so `.textream` files are mainly useful for moving or sharing scripts between machines.

## Building

Requirements:

- Rust with the MSVC toolchain
- Bun
- WebView2
- LLVM / `libclang.dll`
- Visual Studio C++ build tools and CMake

Install LLVM with:

```bash
winget install LLVM.LLVM
```

Then:

```bash
bun install
bun run app
```

Run tests:

```bash
cargo test --workspace
```

Build the installer:

```bash
bun run app:build
```

### Windows CMake note

If CMake tries to select a Visual Studio generator that is not installed, set the generator explicitly before building. For Visual Studio 2022:

```bat
set "CMAKE_GENERATOR=Visual Studio 17 2022"
set "CMAKE_GENERATOR_PLATFORM=x64"
```

## Roadmap

- [x] Platform-neutral prompt engine
- [x] Top-centre prompter overlay
- [x] Hide-from-capture and click-through behavior
- [x] System tray
- [x] Classic mode
- [x] Voice-Activated mode
- [x] Word Tracking with local streaming speech recognition
- [x] Downloadable language models
- [x] Persistent settings
- [x] Font, size, colour and opacity controls
- [x] Pause, hold and mute controls
- [x] Jump to a word / catch up by scrolling
- [x] `.textream` compatibility
- [x] Global shortcuts
- [x] First-run setup and actionable error messages
- [x] Close-to-tray and single-instance behavior
- [x] Crash logging
- [ ] Turkish model support
- [ ] Update check

## Deliberately out of scope

The Windows port is not intended to reach feature parity with the macOS app.

Currently out of scope:

- Remote browser connection / QR pairing
- PowerPoint notes import
- Multi-page editing
- Mirror output for dedicated prompter rigs
- Sidecar-style external display modes

The goal is a focused, solid Windows teleprompter rather than a clone of every macOS feature.
