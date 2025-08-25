# CRC Calculator (GUI) — Requirements

## 1. Overview
A desktop GUI tool to compute and verify CRC checksums for text and files. Supports common CRC presets (CRC‑8/16/32/64) and custom parameterization (polynomial, width, init, reflect in/out, xor‑out). Provides live recalculation, drag‑and‑drop files, streaming for large inputs with progress and cancel, and import/export of CRC profiles.

Primary target: Linux desktop. Architecture: MVVM.

## 2. Goals
- Accurate CRC computation for configurable algorithms.
- Fast, streaming processing of large files without blocking the UI.
- Simple, responsive UI with immediate feedback and validation.
- Easy reuse of CRC settings via presets and user profiles.

Out of scope (v1):
- Network integrations, auto-update, telemetry (optional, disabled by default).
- Non-CRC hash algorithms (MD5/SHA/etc.).

## 3. Users
- Developers/engineers verifying firmware/images/logs.
- QA/testers comparing expected checksums.
- Students learning CRCs.

## 4. Functional Requirements
4.1 CRC Calculation
- Input sources:
  - Text input area (treated as bytes; UTF‑8 by default).
  - File(s) via file picker and drag‑and‑drop.
- Output formats: Hex (uppercase/lowercase), Binary, Decimal.
- Live recalculation while typing in text mode.
- Expected checksum field to compare and show Match/No Match.

4.2 Algorithms and Presets

4.2.1 Algorithm Definition
- Parameters
  - width: Number of CRC bits (1–64).
  - polynomial (poly): Generator polynomial expressed in normal (non‑reflected) form. For refin=true algorithms, the reflected form is used internally.
  - init: Initial register value (width bits).
  - refin (reflectIn): Whether each input byte is bit‑reflected before processing.
  - refout (reflectOut): Whether the final CRC value is bit‑reflected before xorOut.
  - xorOut: Value XORed with the final (optionally reflected) CRC.
- Processing rules
  - The CRC register is width bits; all intermediate results are masked to width.
  - If refin=true, each input byte is reflected (bit‑reversed) before table/bitwise step.
  - If refout=true, reflect the final CRC value (width bits) before applying xorOut.
  - Output formatting (Hex/Bin/Dec) is a presentation concern; the numeric CRC value is defined by the algorithm above.
- Verification vectors
  - “check” value: CRC of ASCII string "123456789" (no newline). Each preset publishes its check for self‑test.
- Implementation
  - Table‑driven fast path for common widths (8/16/32/64) with 256‑entry tables.
  - Bitwise fallback for any width 1–64.
- Validation
  - poly, init, xorOut must fit in width bits; reject if value ≥ 1<<width.
  - width outside 1–64 is invalid.
  - When editing hex fields, accept optional 0x prefix; ignore underscores; enforce width.

4.2.2 Built‑in Presets
- CRC‑8/SMBus
  - width=8, poly=0x07, init=0x00, refin=false, refout=false, xorOut=0x00, check=0xF4
- CRC‑8/MAXIM (Dallas)
  - width=8, poly=0x31, init=0x00, refin=true, refout=true, xorOut=0x00, check=0xA1
- CRC‑16/IBM (ARC)
  - width=16, poly=0x8005, init=0x0000, refin=true, refout=true, xorOut=0x0000, check=0xBB3D
- CRC‑16/MODBUS
  - width=16, poly=0x8005, init=0xFFFF, refin=true, refout=true, xorOut=0x0000, check=0x4B37
- CRC‑16/CCITT‑FALSE
  - width=16, poly=0x1021, init=0xFFFF, refin=false, refout=false, xorOut=0x0000, check=0x29B1
- CRC‑16/X25
  - width=16, poly=0x1021, init=0xFFFF, refin=true, refout=true, xorOut=0xFFFF, check=0x906E
- CRC‑32/ISO‑HDLC (IEEE 802.3)
  - width=32, poly=0x04C11DB7, init=0xFFFFFFFF, refin=true, refout=true, xorOut=0xFFFFFFFF, check=0xCBF43926
- CRC‑32C/Castagnoli
  - width=32, poly=0x1EDC6F41, init=0xFFFFFFFF, refin=true, refout=true, xorOut=0xFFFFFFFF, check=0xE3069283
- CRC‑64/ECMA‑182
  - width=64, poly=0x42F0E1EBA9EA3693, init=0x0000000000000000, refin=false, refout=false, xorOut=0x0000000000000000, check=0x6C40DF5F0B497347

4.2.3 Preset and Profile Behavior
- Built‑in presets are read‑only.
- “Apply Preset” loads parameters into the current session.
- “Save as Profile” creates a user‑editable copy with a custom name and notes.
- Import/Export profiles as JSON. On import, preserve unknown fields for forward‑compatibility.

4.3 File Processing
- Stream-based computation (bounded memory; no full file load).
- Progress indicator (percentage, bytes processed, estimated time).
- Cancellation support; cancels within 200 ms.
- Multiple files: compute individually; per-file results list with copy/save.

4.4 Results and Actions
- Copy checksum to clipboard.
- Save report (text/CSV) including parameters and results.
- Recent items/history (last N inputs/files, clearable).
- Error reporting with actionable messages.

4.5 Validation
- Parameter validation with inline errors and disabled actions until valid.
- Hex input validation for polynomial/init/xorOut.

## 5. Non-Functional Requirements
- Accuracy: Golden vectors for all presets; cross-checked against reference implementations.
- Performance:
  - Process ≥1 GiB file at ≥200 MiB/s on typical modern CPU (release build).
  - Constant memory usage w.r.t. file size.
- Responsiveness:
  - UI remains interactive during long operations.
  - Live text recalculation latency <50 ms for typical inputs (<1 MiB).
- Reliability: Graceful handling of missing files, permissions, and I/O errors.
- Portability: Linux (Wayland/X11). Codebase portable to Windows/macOS.
- Accessibility: Full keyboard navigation; high-contrast friendly.
- Localization: English (en) default; text externalized for future i18n.

## 6. UI Requirements
- Main layout:
  - Left: Input panel (Text area with byte count; File picker + DnD zone; Recent items).
  - Right: Parameters (Preset dropdown; custom params editor with validation).
  - Bottom/Sidebar: Results (Hex/Bin/Dec tabs), Expected vs Actual indicator, progress bar, actions (Copy, Save Report).
- Menu/Toolbar:
  - File: Open, Save Report, Import Profiles, Export Profiles, Exit.
  - Edit: Copy, Clear, Preferences.
  - Help: About.
- Status bar: Operation status, progress, cancel button.

## 7. Data and Formats
- Profile JSON schema:
  - name: string
  - width: uint (1–64)
  - polynomial: uint64 (hex)
  - init: uint64 (hex)
  - reflectIn: bool
  - reflectOut: bool
  - xorOut: uint64 (hex)
  - notes: optional string
- Report: Plain text or CSV including timestamp, inputs, parameters, and outputs.

## 8. Persistence
- Config directory: XDG config (~/.config/CrcCalculator).
- Files:
  - settings.json (UI prefs: theme, last preset, output format).
  - profiles.json (user-defined profiles).
  - history.json (recent files/items; path opt-in).

## 9. Error Handling
- Clear messages for invalid params, unreadable files, and I/O failures.
- Non-blocking dialogs; logs available in a diagnostic pane.

## 10. Testing and QA
- Unit tests:
  - CrcEngine vector tests per preset; custom parameter edge cases.
  - Reflection and xorOut correctness; width boundaries (1, 64).
- Property-based/fuzz tests for random parameters against a slow reference.
- Integration tests:
  - File streaming with cancellation and progress.
  - ViewModel command flows.
- Manual test checklist:
  - DnD, live typing, profile import/export, large file (≥5 GiB) cancel/resume.

## 11. Build and Tooling
- Language: C++23.
- Build: CMake; Release/Debug configs, sanitizers in Debug.
- Dependencies:
  - GUI: Qt 6 (recommended) or equivalent with MVVM-capable bindings.
  - JSON: nlohmann/json or Qt JSON.
- Packaging helpers: CPack (optional), AppImage tooling (linuxdeploy, appimagetool).

## 12. Security & Privacy
- No network access by default.
- File paths/history stored locally; opt-out setting to disable history.
- No telemetry by default.

## 13. Acceptance Criteria
- Compute correct checksums for built-in presets (all golden tests pass).
- Live text updates and clipboard copy work.
- Large file processing shows progress and supports cancel.
- Profiles can be created, saved, imported, exported.
- UI responsive under load; no crashes in basic workflows.

## 14. Deployment

14.1 Packaging Strategy (Simple by default)
- Primary format: AppImage (single executable) containing the app and all required Qt/runtime libraries.
- Optional formats:
  - .deb via CPack for Debian/Ubuntu users.
  - Flatpak manifest for future Flathub distribution (sandboxed; minimal permissions).

14.2 End‑User Install Experience
- AppImage:
  - Install: download → chmod +x → run. No root required. First run may offer desktop integration (menu entry, icon).
  - Uninstall: delete the AppImage file. If integrated, remove generated desktop/icon files or use the integration tool’s “remove” option.
- .deb:
  - Install: sudo apt install ./CrcCalculator_VERSION_amd64.deb or via repository if provided.
  - Uninstall: sudo apt remove crc-calculator.
- Flatpak (future):
  - Install: flatpak install flathub org.example.CrcCalculator.
  - Uninstall: flatpak uninstall org.example.CrcCalculator.

14.3 Distribution Channels
- GitHub Releases:
  - Assets: AppImage, SHA256 checksum file, GPG signature (optional), LICENSE, README, CHANGELOG.
  - Release notes summarize changes and known issues.
- Optional:
  - APT repository/PPA for .deb.
  - Flathub for Flatpak when ready.

14.4 Compatibility Requirements
- Architecture: x86_64 Linux.
- glibc baseline: build on Ubuntu 20.04 to maximize compatibility across distros.
- GPU: works with OpenGL or Qt’s software rendering fallback.
- No system Qt required for AppImage; all dependencies must be bundled.

14.5 Security, Integrity, and Trust
- Publish SHA256 checksums for each artifact.
- Optionally sign AppImage and checksum file with a GPG release key; publish fingerprint in README.
- Users can verify:
  - sha256sum CrcCalculator-*.AppImage
  - gpg --verify CrcCalculator-*.AppImage.sig CrcCalculator-*.AppImage

14.6 Updates and Rollback
- AppImage: manual updates by downloading newer file; older versions remain usable (side‑by‑side). Optional .zsync for delta updates.
- .deb: updates delivered via apt if a repo is provided; support rollback to previous version via package manager.
- Keep at least the last two minor versions available in Releases with a clear CHANGELOG.

14.7 Size and Performance Constraints
- AppImage target size: ≤ 100 MB if feasible (strip binaries; remove unused Qt modules).
- Startup time: ≤ 1 s on a typical desktop (cold start).
- Runtime must function fully offline; no network access required.

14.8 Desktop Integration and UX
- Bundle a .desktop file, 256×256 icon, and MIME type for profile files (*.crcprofile.json).
- AppImage/packaging must ensure:
  - Proper desktop entry in menus.
  - File associations for profile import (optional).
  - Drag‑and‑drop works without additional system packages.

14.9 Privacy and Offline Use
- No telemetry and no outbound network calls.
- History and settings stored under XDG directories in the user home; easy opt‑out for history.

14.10 Release Requirements
- Use Semantic Versioning (MAJOR.MINOR.PATCH).
- Name artifacts consistently: CrcCalculator-${VERSION}-x86_64.AppImage.
- Each release must include: artifacts, checksums, LICENSE, and CHANGELOG.
