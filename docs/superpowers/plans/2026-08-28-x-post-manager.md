# X Post Manager Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a standalone Windows X Post Manager v1.0.0 that connects multiple X accounts, queues and schedules distinct videos, composes manual captions plus a reusable suffix, prepares oversized video, publishes through X API v2, and self-updates from this repository.

**Architecture:** A lightweight Tkinter desktop shell calls isolated services for SQLite persistence, scheduling, video preparation, OAuth/token storage, X API upload/posting, and updates. The app is packaged as a PyInstaller one-directory distribution with a separate updater executable and a dedicated GitHub Actions release pipeline.

**Tech Stack:** Python 3.12, Tkinter/ttk, sqlite3, requests, Windows DPAPI via ctypes, ffmpeg/ffprobe, PyInstaller, unittest.

**Spec:** `docs/superpowers/specs/2026-08-28-x-post-manager-design.md`

## Global Constraints
- Windows 10/11 x64.
- X website automation is forbidden; use official X API only.
- OAuth scopes: `tweet.read tweet.write users.read media.write offline.access`.
- Telegram Admin Manager 4.2.9 files and release workflow must remain behaviorally unchanged.
- X Post Manager version starts at `1.0.0` and uses release tags `x-v<version>`.
- Standard safe video target: 135 seconds and 500 MB per part; hard standard limit 140 seconds and 512 MB.
- No X access/refresh token may be written to logs or stored unencrypted on Windows.

---

### Task 1: Core models, caption composition, persistence

**Files:**
- Create: `x-post-manager/xpostmanager/__init__.py`
- Create: `x-post-manager/xpostmanager/models.py`
- Create: `x-post-manager/xpostmanager/captions.py`
- Create: `x-post-manager/xpostmanager/db.py`
- Test: `x-post-manager/tests/test_captions.py`
- Test: `x-post-manager/tests/test_db.py`

**Interfaces:**
- Produces: `compose_post_text(caption: str, suffix: str) -> str`
- Produces: `Database(path).add_account(...)`, `add_job(...)`, `list_jobs(...)`, `update_job_status(...)`

- [ ] Write failing caption tests for empty/non-empty caption/suffix combinations and exact blank-line joining.
- [ ] Run `python -m unittest tests.test_captions -v` and verify failure because production module is absent.
- [ ] Implement `compose_post_text` minimally and rerun until green.
- [ ] Write failing database tests for schema creation, job persistence, and status transition persistence across reopen.
- [ ] Implement SQLite schema and repository methods; rerun `python -m unittest tests.test_db -v` until green.

### Task 2: Scheduling and round-robin distribution

**Files:**
- Create: `x-post-manager/xpostmanager/scheduling.py`
- Test: `x-post-manager/tests/test_scheduling.py`

**Interfaces:**
- Produces: `build_schedule(video_ids, account_ids, start_date, daily_times, per_account_per_day) -> list[Assignment]`
- Produces: `due_jobs(jobs, now) -> list[Job]`

- [ ] Write failing tests proving each account receives distinct videos, four configured slots are filled chronologically, and distribution continues on following days.
- [ ] Run scheduling tests and verify expected failures.
- [ ] Implement deterministic round-robin distribution with timezone-naive local schedule timestamps stored in ISO format.
- [ ] Add due-job tests and minimal implementation; run full scheduling suite green.

### Task 3: Video probing and segmentation plan

**Files:**
- Create: `x-post-manager/xpostmanager/video.py`
- Test: `x-post-manager/tests/test_video.py`

**Interfaces:**
- Produces: `probe_video(path, ffprobe_path) -> VideoInfo`
- Produces: `plan_segments(info, target_seconds=135, target_bytes=500*1024*1024) -> list[SegmentPlan]`
- Produces: `prepare_video(path, output_dir, ffmpeg_path, ffprobe_path) -> list[PreparedVideo]`

- [ ] Write failing pure planning tests: 100-second video produces one segment; 300-second video produces three; oversized short file marks transcode required.
- [ ] Run tests and verify red.
- [ ] Implement the pure planner and make tests green.
- [ ] Add command-construction tests for ffprobe, stream-copy splitting, and H.264/AAC fallback transcoding without executing ffmpeg.
- [ ] Implement subprocess runner with progress callbacks and safe output filenames.

### Task 4: DPAPI token vault, OAuth 2.0 PKCE, X API client

**Files:**
- Create: `x-post-manager/xpostmanager/secrets.py`
- Create: `x-post-manager/xpostmanager/oauth.py`
- Create: `x-post-manager/xpostmanager/xapi.py`
- Test: `x-post-manager/tests/test_oauth.py`
- Test: `x-post-manager/tests/test_xapi.py`

**Interfaces:**
- Produces: `TokenVault.protect(str) -> bytes`, `unprotect(bytes) -> str`
- Produces: `OAuthPkce.start_authorization(client_id, redirect_uri) -> AuthorizationSession`
- Produces: `OAuthPkce.exchange_code(...) -> TokenSet`, `refresh(...) -> TokenSet`
- Produces: `XApiClient.upload_video(path, token, progress_cb) -> media_id`
- Produces: `XApiClient.create_post(text, media_id, token) -> post_id`

- [ ] Write failing PKCE tests for verifier/challenge generation and state validation.
- [ ] Implement PKCE helpers and exchange/refresh request construction using an injectable HTTP transport.
- [ ] Write failing X API tests for initialize/append/finalize/status sequence and create-post JSON.
- [ ] Implement chunked media upload with bounded retry/backoff and status polling.
- [ ] Add tests proving auth headers/tokens are never included in raised/loggable error text.

### Task 5: Background scheduler and job execution

**Files:**
- Create: `x-post-manager/xpostmanager/worker.py`
- Test: `x-post-manager/tests/test_worker.py`

**Interfaces:**
- Produces: `JobRunner.run_job(job_id) -> None`
- Produces: `SchedulerService.start()`, `pause()`, `stop()`

- [ ] Write failing state-machine tests for `queued -> preparing -> ready -> uploading -> publishing -> published`.
- [ ] Write failure-path test proving API failure results in `failed`, never `published`.
- [ ] Implement `JobRunner` with persisted transitions and injectable video/API services.
- [ ] Implement scheduler loop that selects due jobs and prevents duplicate simultaneous execution of one job.

### Task 6: Desktop UI and tray behavior

**Files:**
- Create: `x-post-manager/xpostmanager/app.py`
- Create: `x-post-manager/xpostmanager/ui.py`
- Create: `x-post-manager/xpostmanager/tray.py`
- Create: `x-post-manager/run_x_post_manager.py`
- Test: `x-post-manager/tests/test_ui_logic.py`

**Interfaces:**
- UI consumes Database, SchedulerService, OAuthPkce, and video import service.

- [ ] Write failing non-GUI tests for queue-row formatting, status summary counts, and caption preview model.
- [ ] Implement dashboard window with Accounts panel, Queue table, buttons for Add videos/Edit caption/Auto distribute/Schedule/Start-Pause/Retry.
- [ ] Implement dialogs for Client ID + reusable suffix, account authorization, daily slots, and per-item caption.
- [ ] Implement tray icon so closing/minimizing can leave the scheduler running; add explicit Exit command.
- [ ] Add first-run onboarding that clearly states a Developer App Client ID is required and provides the exact callback URI `http://127.0.0.1:8765/callback`.

### Task 7: Self-updater

**Files:**
- Create: `x-post-manager/xpostmanager/update.py`
- Create: `x-post-manager/xpostmanager/updater_main.py`
- Create: `x-post-manager/update.json`
- Create: `x-post-manager/release.json`
- Test: `x-post-manager/tests/test_update.py`

**Interfaces:**
- Produces: `parse_manifest(json_text) -> UpdateManifest`
- Produces: `is_newer(current, candidate) -> bool`
- Produces updater CLI: `XPostManagerUpdater.exe --manifest-url ... --install-dir ... --pid ...`

- [ ] Write failing manifest/version/SHA verification tests.
- [ ] Implement manifest parser, semantic version comparison, download and SHA-256 verification.
- [ ] Implement separate updater process that waits for parent PID, extracts to staging, replaces application files, and relaunches `XPostManager.exe`.
- [ ] Wire update check into UI, showing both available version and notes before update.

### Task 8: Packaging, ffmpeg bundle, GitHub release workflow

**Files:**
- Create: `x-post-manager/requirements.txt`
- Create: `x-post-manager/XPostManager.spec`
- Create: `x-post-manager/XPostManagerUpdater.spec`
- Create: `x-post-manager/installer.iss`
- Create: `.github/workflows/publish-x-post-manager.yml`
- Create: `x-post-manager/README.md`

**Interfaces:**
- Workflow produces `XPostManager_v<version>_update.zip` and `XPostManager_Setup_v<version>.exe`, then publishes `x-post-manager/update.json` last.

- [ ] Add pinned runtime/build dependencies and PyInstaller specs.
- [ ] Configure Windows workflow to run `python -m unittest discover -s x-post-manager/tests -v` before packaging.
- [ ] Download a Windows ffmpeg essentials build during CI and copy `ffmpeg.exe`/`ffprobe.exe` into the packaged app directory.
- [ ] Build app and updater with PyInstaller one-directory mode.
- [ ] Build installer with Inno Setup.
- [ ] Create/update GitHub release tag `x-v${VERSION}`, upload both assets, compute ZIP SHA-256, then update `x-post-manager/update.json` and push the manifest.
- [ ] Verify release contains both assets and manifest version/SHA match.

### Task 9: End-to-end offline verification

**Files:**
- Modify tests as needed only to cover real interfaces; no live credentials.

- [ ] Run `python -m compileall -q x-post-manager`.
- [ ] Run `python -m unittest discover -s x-post-manager/tests -v` and require zero failures.
- [ ] Validate both JSON manifests with Python json parser.
- [ ] Review logs/error formatting for token leakage.
- [ ] Confirm Telegram `release.json`, `update.json`, and `publish-release.yml` are unchanged by comparison with branch base.
