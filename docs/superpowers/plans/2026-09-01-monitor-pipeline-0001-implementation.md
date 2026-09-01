# Monitoring Pipeline and 0001 Version Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship Telegram Admin Manager 0001 so idle accounts can start the next waiting monitoring rule while the previous rule finishes its last in-flight files, and migrate visible versions to four-digit serials.

**Architecture:** Consecutive monitoring queue items are executed as one shared-client pipeline wave. Tasks retain priority by releasing task N+1 only after task N has no unassigned queued files; existing account locks/global semaphore remain the authority for actual concurrency. The updater gains a compatibility/display version split so legacy 4.3.8 clients can install v0001 without serial clients seeing an update loop.

**Tech Stack:** Python 3, asyncio, Tkinter/ttk, Telethon 1.44.0, unittest, GitHub Actions release-patch pipeline.

**Spec:** `docs/superpowers/specs/2026-09-01-monitor-pipeline-version-0001-design.md`

## Global Constraints

- Base release: 4.3.8.
- Visible target version/tag/package: 0001.
- Preserve one long-lived Telethon client per session and one active upload per session.
- Preserve global parallel, Premium routing, FloodWait, Telegram restriction status, FilePartMissingError tail requeue, captions, ETA, pause/stop and upload history.
- Pipeline only consecutive monitoring-origin queue items; do not reorder across manual items.
- Do not merge queue items that contain overlapping task IDs.

---

### Task 1: Queue-wave collection and status

**Files:**
- Modify: `batch_uploader.py`
- Test: `tests/test_0001_monitor_pipeline.py`

- [ ] Add failing tests for consecutive-monitor collection, stop at manual item, overlap stop and original-order preservation.
- [ ] Implement pure helpers for monitor-wave selection.
- [ ] Add `current_batch_ids` so multiple queue rows can be represented as active.
- [ ] Update queue summary/removal guards for multiple active items.
- [ ] Run focused tests.

### Task 2: Priority release between monitoring tasks

**Files:**
- Modify: `batch_uploader.py`
- Test: `tests/test_0001_monitor_pipeline.py`

- [ ] Add failing source/behavior tests for per-task start/drained gates.
- [ ] Extend `_run_batch(..., pipeline=False)` to build ordered start/drained events only for pipeline waves.
- [ ] Extend `_run_one_task(..., start_after=None, drained_event=None)` so it prepares queues, waits for predecessor release, and signals release when its unassigned queues are empty; always signal on early return/finally to avoid deadlock.
- [ ] Preserve account locks/global semaphore as actual concurrency control.
- [ ] Run focused and existing upload/monitor tests.

### Task 3: Multi-item queue execution/finalization

**Files:**
- Modify: `batch_uploader.py`
- Test: `tests/test_0001_monitor_pipeline.py`

- [ ] Modify `_queue_worker` to collect a monitoring wave, resolve all distinct tasks in queue order and call one `_run_batch(..., pipeline=True)`.
- [ ] Mark each original queue row/db item running and finalize each independently from its task IDs.
- [ ] Persist each item's monitoring signature only when its own tasks succeed.
- [ ] On Stop reinsert original wave items in original order.
- [ ] Keep non-monitoring queue execution single-item/legacy.
- [ ] Run focused tests and full regression suite.

### Task 4: Serial version compatibility

**Files:**
- Modify: `telegram_admin_manager.py`
- Modify: `tam_core/updater.py`
- Test: `tests/test_0001_updater_serial_versions.py`

- [ ] Add failing tests for legacy 4.3.8 -> manifest 4.3.8.0001/display 0001, installed 0001 no loop, and 0001 -> display 0002.
- [ ] Update manifest parsing to compare legacy installs by compatibility `version` and serial installs by `display_version`.
- [ ] Return/display/install the serial version from `display_version`.
- [ ] Set `APP_VERSION = "0001"`.
- [ ] Run focused updater tests.

### Task 5: Release workflow and publication

**Files:**
- Modify: `.github/workflows/publish-release.yml`
- Modify: `release.json`
- Add: `release-src/0001/...` patch files

- [ ] Extend workflow manifest publication to use optional `manifest_version` and `display_version` while tag/asset/package naming uses `version`.
- [ ] Set release version `0001`, base `4.3.8`, manifest compatibility `4.3.8.0001`, display `0001`.
- [ ] Run full `unittest`, compileall and GUI construction smoke locally on modified source.
- [ ] Generate patch from clean published 4.3.8 and verify patch application plus full tests on rebuilt candidate.
- [ ] Open PR and require candidate CI to rebuild from v4.3.8, test and build `TelegramAdminManager_v0001_update.zip`.
- [ ] Merge only on green candidate CI.
- [ ] Verify final v0001 release, asset SHA and `update.json` containing compatibility `version=4.3.8.0001` and `display_version=0001`.
