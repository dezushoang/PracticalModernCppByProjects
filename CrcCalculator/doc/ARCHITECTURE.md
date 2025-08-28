# CRC Calculator (GUI) — Architecture Specification (arc42)

Author: Dezus  
Version: 1.0  
Status: Released  
Scope: Desktop GUI app for CRC calculation (Linux-first), implemented with C++23 + Qt 6, MVVM.

References:
- Requirements: ./REQUIREMENT.md

## 1. Introduction and Goals
- Business goals: Accurate, fast CRC computation; responsive GUI; reusable presets/profiles; large-file streaming with progress and cancel.
- Quality goals (top-3):
  1) Correctness: Golden vectors, reproducible results across presets and custom params.  
  2) Performance & Responsiveness: ≥200 MiB/s file throughput; UI stays interactive; live updates <50 ms.  
  3) Reliability: Graceful IO errors; cancellation within 200 ms; no crashes in basic workflows.
- Stakeholders: End users (dev/test/QA/students), maintainers, packagers.

## 2. Constraints
- Platform: Linux (Wayland/X11); portable to Windows/macOS.
- Tech: C++23, Qt 6 (MVVM-friendly bindings), nlohmann/json or Qt JSON.
- Packaging: AppImage primary; optional .deb; future Flatpak.
- No network by default; XDG base dirs for persistence.
- One desktop process; constant memory with respect to file size.

## 3. Context and Scope
The app runs locally, interacts with filesystem and clipboard, stores configs under XDG, and renders via Qt.

Diagram: see diagrams/context.puml  
Key external interfaces:
- OS/Window system (Qt platform plugin), Clipboard.
- File system (read inputs, save reports).
- XDG config directory (~/.config/CrcCalculator).

## 4. Solution Strategy
- MVVM with Qt: Views (Qt Widgets/QML), ViewModels (QObject with properties/signals), Models/Services (pure C++ or Qt-enabled).
- CRC Engine: Table-driven fast path (8/16/32/64, 256-entry tables) + bitwise fallback for width 1–64. Reflection controls per algorithm.
- Streaming: FileProcessor reads in chunks (e.g., 1–8 MiB), updates CRC incrementally, emits progress via signals, supports cancel via atomic flag.
- Concurrency: Single GUI thread + worker threads (QThreadPool/QtConcurrent) for file processing; progress marshaled to GUI thread.
- Validation: Strict parameter and input validation; actions disabled until valid.
- Persistence: JSON files under XDG; tolerant readers preserving unknown fields.

## 5. Building Block View (Components)
Layers:
- UI (Views): MainWindow, panels (Input, Parameters, Results).
- ViewModels: MainViewModel, ParametersVM, TextInputVM, FileJobsVM, ResultsVM.
- Domain/Services: CrcEngine, CrcTable, ByteParsers, PresetsProvider, ProfileStore, SettingsStore, HistoryStore, FileProcessor, ReportWriter(s).
- Infra: JSON serializer, XDG paths, logging, error mapper.

Diagram: see diagrams/components.puml

Key responsibilities:
- CrcEngine: Parameterized CRC compute; exposes table and bitwise paths.
- FileProcessor: Stream file, handle progress/ETA, cancellation, backpressure-free updates.
- ByteParsers: Parse String/Hex/Binary text to bytes with diagnostics.
- ProfileStore: Import/export profiles.json (preserve unknown fields).
- ResultsVM: Format output Hex/Bin/Dec, match against expected.

Interfaces (selected):
- IFileJob: start(file), cancel(), progressSignal(bytes,total,eta), completedSignal(result|error).
- ICrcEngine: compute(bytes), update(state, chunk), finalize(state).
- IProfileStore: loadAll(), saveAll(), import(json), export(json).

## 6. Runtime View (Key Scenarios)
- Text live CRC:
  - User types → TextInputVM validates/parses → CrcEngine computes → ResultsVM formats → View updates; debounced at frame-rate if needed.
- File CRC:
  - User drops file → FileJobsVM creates job → FileProcessor runs in worker → emits progress → ResultsVM updates progress and final result; cancel honored ≤200 ms.
- Profiles:
  - Apply preset → ParametersVM updates → CrcEngine self-test (check value) optional quick verify → ResultsVM recomputes.

Diagrams:
- diagrams/sequence_text_input.puml
- diagrams/sequence_file_crc.puml

## 7. Concurrency and Threading
- GUI Thread: All UI and ViewModel property changes; signal/slot queued connections from workers.
- Worker Pool (QThreadPool): FileProcessor jobs (I/O + CRC compute); one thread per job by default; can throttle.
- Cancellation: std::atomic<bool> checked per-chunk; QFile read loops observe flag; job tears down cooperatively.
- Progress cadence: throttled to ~10–20 Hz to reduce UI churn while meeting responsiveness goals.

Diagram: diagrams/threads.puml

## 8. Deployment View
- Single binary with Qt libs (AppImage).
- Optional platform plugins for rendering; uses software fallback if GPU absent.
- No external services; all offline.
- Build on Ubuntu 20.04 for glibc baseline.

## 9. Cross-cutting Concepts
- Validation and Error Reporting: Typed error codes mapped to user-friendly messages; inline validation disables actions until fixed.
- Data formatting: Hex upper/lower case switch; binary and decimal outputs; width-aware masking.
- Logging: Debug build logs to console; optional rolling file logger in diagnostics pane.
- Internationalization-ready: Strings externalized; default en.
- Accessibility: Focus traversal, high-contrast palette.

## 10. Architectural Decisions (Summary)
- AD-1 MVVM with Qt vs MVC/MVP: MVVM aligns with Qt properties/bindings; reduces glue code.
- AD-2 Qt worker pool vs std::thread: Use QThreadPool/QtConcurrent for safe signal marshaling to GUI; simplifies cancel/progress.
- AD-3 Table-driven CRC for 8/16/32/64: Meeting throughput goals; bitwise fallback keeps full width range.
- AD-4 JSON library: Prefer nlohmann/json for ergonomics; Qt JSON acceptable alternative to reduce deps.
- AD-5 AppImage primary packaging: Simplest UX, bundles Qt; meets offline requirement.

Alternatives in §14.

## 11. Quality Requirements (Scenarios)
- Performance: 1 GiB file processed ≥200 MiB/s on typical CPU. Budget: I/O ≥1.5 GB/s peak; compute ≥2 GB/s for CRC-32 table path; progress <= 15% overhead.
- Responsiveness: Live text <50 ms end-to-end for <1 MiB; progress updates 50–100 ms cadence; cancel ≤200 ms.
- Reliability: Missing file → non-blocking error; partial reads handled; history opt-in for paths.

Timing diagram: diagrams/timing_large_file.puml

## 12. Data View
- Profile JSON schema (see REQUIREMENT.md §7). Unknown fields preserved on round-trip.
- settings.json: UI prefs (theme, last preset, output format).
- history.json: recent files/items, opt-in for paths.
- Reports: text or CSV with timestamp, parameters, outputs.

## 13. Error Handling
- Categorized: ValidationError, IoError, ParseError, ComputeError, Cancelled.
- UI behavior: Inline errors; disabled actions; non-blocking dialogs; diagnostic pane with details and remediation hints.

## 14. Alternatives and Trade-offs
- Pattern:
  - MVVM (chosen) vs MVP/MVC: MVVM maps to Qt’s property system and QML/Widgets; MVP needs more presenter plumbing; MVC blurs responsibilities in client GUI.
- GUI Toolkit:
  - Qt (chosen) vs wxWidgets/GTK/ImGui: Qt has mature MVVM, bindings, cross-platform, rich packaging ecosystem; ImGui unsuitable for native desktop look; wx/GTK weaker on MVVM patterns and packaging.
- Concurrency:
  - QtConcurrent/QThreadPool (chosen) vs std::thread + custom queues: Qt simplifies cross-thread signals and lifetime; std::thread adds boilerplate and risks UI thread misuse.
- CRC Implementation:
  - Table-driven + bitwise fallback (chosen) vs solely bitwise: Chosen meets throughput goals; fallback covers arbitrary widths without excessive memory.
  - Precompute-once-per-params (chosen) vs per-call: amortizes cost across file chunks; friendlier to streaming.
- Packaging:
  - AppImage (chosen primary) vs Flatpak vs .deb: AppImage simplest; Flatpak sandboxing is heavier; .deb good for repo users but not self-contained.

## 15. Risks and Mitigations
- Large-file throughput varies by disk: Allow adjustable chunk size; asynchronous I/O possible later.
- GUI freezes from excessive signal spam: Throttle progress; coalesce updates.
- Parameter mistakes cause wrong CRCs: Golden vector self-test per preset; unit/property tests for edge widths (1, 64).
- Cancellation races: Check atomic flag between read/compute steps; use queued connections for completion.

## 16. Testing Strategy
- Unit: CrcEngine vectors for all presets; reflection/xorOut; width bounds; ByteParsers.
- Property/Fuzz: Random parameter sets versus slow reference.
- Integration: File streaming with cancel and progress; ViewModel command flows.
- Manual: DnD, live typing, profile import/export, ≥5 GiB cancel/resume.

## 17. Workflows (How-to)
- Live text: Validate → parse → compute → format → display; disabled compute on invalid input.
- Files: Add job → worker stream compute → periodic UI progress → final result; cancel -> signal job to stop and mark cancelled quickly.

## 18. Glossary
- CRC width: number of bits of the CRC register.
- refin/refout: reflection of input bytes/final CRC respectively.
- xorOut: value XORed at the end.
- Preset: read-only, known-good parameter set.
- Profile: user-defined, import/exportable set of parameters.

---
Diagrams are stored under ./diagrams. Open .puml files with VS Code PlantUML extension or render via `plantuml`.