# Changelog

All notable changes to Agora, newest first. Dates are the day the work landed.

## 2026-10-17 — publish readiness: a clone that builds, and a current dependency set

Preparing the project to stand alone surfaced three defects that a fresh clone would
have hit. All three are fixed and verified.

- **A fresh clone did not build.** `.gitignore` carried `*.g.dart`, so
  `lib/data/database.g.dart` was never committed. Proven by simulating a clone
  (`rsync` the package to a scratch directory and delete every generated file):
  `flutter analyze` then reported **297 issues**. `lib/data/database.g.dart`
  (8149 lines) is now tracked and the blanket ignore is gone. It is committed **on
  purpose** so a clone builds and tests with no codegen step; regenerate it only after
  a schema change with `dart run build_runner build --delete-conflicting-outputs`.
  The scratch clone now analyzes clean and its full suite passes.
- **A machine-specific path was committed.** `tool/sandbox/adapter_sandbox.json`
  hard-coded an absolute `file:///home/…/tool/sandbox/…` URL. It now ships the relative
  `file://./tool/sandbox/mock_ai_site.html`, and `AdapterConfig.fromJson` expands a
  relative `file://` URL against the working directory at parse time — `http`,
  `about:` and absolute `file:///` URLs pass through untouched. The new
  `test/adapter_config_test.dart` covers the expansion, the parent-relative case, the
  pass-through cases and the shipped fixture. No tracked file contains a machine path
  any more.
- **A dependency was discontinued.** `flutter pub upgrade` reported *"1 package is
  discontinued"*; it was `flutter_markdown 0.7.7+1`, replaced upstream by
  `flutter_markdown_plus`. Both call sites migrated (`decision_record_screen.dart`,
  `transcript_view.dart`) — it is a drop-in fork, so no widget code changed — and the
  full upgrade was taken (29 packages). The warning is gone.
- **After the upgrade:** `flutter analyze` **0 issues**; `flutter test` **296 passed /
  1 skipped** (up from 290; the new `adapter_config_test.dart` adds 6).

## 2026-10-17 — AppImage: a FUSE-free artifact and a real launch proof

The reported failure was `Cannot mount AppImage, please check your FUSE setup.` It was
diagnosed rather than guessed at, and the packaging script now ships a second artifact.

- **Not a missing library.** `libfuse.so.2` is present, `/dev/fuse` exists, and
  `/usr/bin/fusermount3` is setuid root. The cause is `NoNewPrivs=1` in the
  environment, which nullifies the setuid bit; Ubuntu 24.04's
  `kernel.apparmor_restrict_unprivileged_userns=1` breaks it the same way outside a
  container.
- **`tool/build_appimage.sh` now emits a portable tarball** alongside the AppImage
  (`build/agora-0.1.0-x86_64.tar.gz`). A type-2 AppImage that cannot mount cannot start
  at all; the tarball is the same bundle and needs nothing beyond the libraries beside
  the binary (`$ORIGIN` RPATH). It is the only runnable form on a locked-down host.
- **Payload proved by extraction:** `--appimage-extract` → a complete `squashfs-root`
  containing `AppRun`, the `agora` ELF pie executable, `data/`, `lib/` and `usr/`.
- **Launch proved without FUSE:** `xvfb-run -a ./AppRun` ran the full 30 s timeout (exit
  124 — alive, not crashed), logged the Impeller/OpenGLES backend, and created
  `~/.config/agora/agora.sqlite` at `user_version 3` with all 14 tables including the
  FTS5 set. The tarball path and `APPIMAGE_EXTRACT_AND_RUN=1` were verified identically.
- Closes a gap: launch had been unrepeatable since FUSE was denied, and was last
  verified on 2026-10-10.

## 2026-10-17 — the Linux build was never actually blocked

Both `tool/cpp_lint.sh` and `flutter build linux` had been recorded as blocked since
2026-10-06. Neither blocker was real.

- `tool/cpp_lint.sh` → **PASS**, 5 files, 0 errors, 0 warnings, exit 0. The
  `libgtk-3-dev` / `libglib2.0-dev` absence no longer reproduces; `pkg-config gtk+-3.0`
  resolves GTK 3.24.52 and glib 2.88.0.
- `flutter build linux --release` → **PASS**. Its `cmake` traceback was never a missing
  package: a stale `pip install --user` shim at `~/.local/bin/cmake`, installed under
  Python 3.12 (the system has since moved to 3.14), still resolved on `PATH` but exited
  with `ModuleNotFoundError`, shadowing a working `/usr/bin/cmake 4.2.3`. `ninja` was
  suspected too, but it is a real ELF binary and always ran. `tool/build_appimage.sh`
  now detects a tool that resolves but does not run, falls back to `/usr/bin`, and
  prints a note instead of dying — which is exactly the situation a contributor on a
  machine with an upgraded Python can hit, so the guard stays.
- `appimagetool` is itself an AppImage and needs FUSE; where mounts are denied the
  script retries with `APPIMAGE_EXTRACT_AND_RUN=1` after a FUSE failure.
- Rebuilt `build/agora-0.1.0-x86_64.AppImage` — **12 MB**.

## 2026-10-17 — analyzer and lint cleanup

`dart analyze` on the package went from **20 errors + 197 info (exit 3)** to
**`No issues found!` (exit 0)**.

- Every one of the 20 errors was in gitignored `tool/` probe scratch
  (`check_load*.dart`, `debug3.dart`, `debug_connect*.dart`, `probe_fix.dart`) — dead
  helpers written against an abandoned API. The scratch is kept on disk by decision, so
  `analysis_options.yaml` now excludes **the same globs as `.gitignore`**. That is a
  scope fix, not a suppression: the files are not part of a checkout, and the analyzer
  was the only thing still policing them. `dart analyze $(git ls-files 'tool/*.dart')`
  remains the honest check for `tool/`.
- `dart fix --apply` cleared 34 findings; the remaining 24 were fixed by hand:
  `print` → `stdout.writeln` in `tool/focus_probe.dart` (12) and
  `tool/registry_check.dart` (4); `_isLeader` plus getter/setter collapsed to one
  mutable `bool isLeader` field in `lib/core/orchestration/orchestrator.dart`;
  `_HOME_NOTE` → `_homeNote`; `r_scoreSeries` → `rScoreSeries`; `FakeAgent.close()`
  now closes the two `StreamController`s it owns; two `exists()` → `existsSync()` in
  `lib/core/adapters/adapter_registry.dart`.
- **Genuine bug fixed.** The Verbatim hash check in `tool/sandbox_smoke.dart` compared
  a `String?` to a `bool`, so it was vacuously true and asserted nothing. It now asserts
  `contentHash == contentHash(rawText)`, which is the guarantee
  `SessionRepository.isIntact` relies on.
- `flutter test` 290 passed / 1 skipped at the time; `flutter analyze` clean.

## 2026-10-17 — documentation brought under a standard

- Added `docs/README.md` and this changelog, and re-verified `STATE.md`.
- Documentation was reorganised so one concern lives in one file, and every top-level
  document is indexed from `docs/README.md`.

## 2026-10-06 — Amendment R1 ratified

Closes the dispatch contradiction that had blocked `SpeakerSelector`. Round-robin was
adopted, but as `broadcast` under the approved plan rather than as `PLAN-GROUPCHAT.md`
D1; it supersedes §2.4, §8, §11 and §17 where they conflict.

- Protocol parser for all five actions, the orchestrator sender table, and
  channel-aware `MessageCaptured` / `PacketDispatched`.
- drift v3: `channels`, `messages.channel_id`, and repository CRUD.
- Per-thread journaling in `CouncilRunner`; the centre pane reads the selected thread.
- Slash-command composer; silent Chrome mode with reveal; conversation chips carrying
  avatar stacks, presence dots and unread badges (read state is a per-thread message
  count, not a timestamp — drift's `createdAt` is whole seconds).
- Convergence card and graph.
- Added `tool/live_check.dart` (reads a real page and dumps its text) and
  `tool/page_text.dart`.

## 2026-10-03 — the build-out

The day Agora went from scaffold to a working app. Highlights:

- **UI mounted** — three-pane council room, `@`-routing composer, calibration studio,
  decision record, settings.
- "Start the council" runs for real, with live progress, a stop button and a
  blocked-reason card; real snapshot (Ctrl+S) and interrupt (Esc).
- `TabPool` + `CouncilRunner`; the orchestrator state machine; the 57-test
  orchestration suite.
- `SessionRepository`; schema v2 plus hot-query indexes, and the index-DDL fixes that
  followed.
- Prompt compiler (B1 delta packets, D1 brief) plus the safety firewall.
- `SelectorProber` — ask a real page which selector tier actually resolved.
- Time Machine — journal dispatch and capture, and replay against the Sandbox Lab.
- Onboarding wizard and `LoginChecker`.
- Defeated background-tab throttling.
- AppImage packaging — desktop entry, metainfo, icon, `AppRun`, build script.

## 2026-09-30 — the adapter framework

- Adapter framework: a data-driven config model plus the JS page kernel.
- Real adapter configs for ChatGPT / Gemini / DeepSeek (3/3 validate).
- Adapter registry — bundled plus user JSON overlay, hot reload, graceful skip.
- Adapter config contract, drift codegen, agent and web adapters.
- Sandbox Lab — mock AI site fixture plus adapter end-to-end smoke test.
- Sandbox suite green: **42/42** e2e against real Chrome.
- Critical fix: capture the answer baseline *before* the send click.

## 2026-09-28 — scaffolding

- P0 baseline: clean Flutter Linux desktop scaffold.
- P1 CDP engine, proven against real Chrome 154.
- Council Protocol parser plus 24 passing tests.
- Restructured `story-app` into Agora.
