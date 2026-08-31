# Telegram Admin Manager 4.3.3 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship 4.3.3 with live caption updates, unified monitoring, persistent right-side logs, persistent FloodWait cooldowns, and maximum supported admin rights.

**Architecture:** Keep the durable queue and existing state database as internal infrastructure while making monitoring rules the single user-facing workspace. Resolve mutable rule data at the latest safe boundary before Telegram sends, and persist cooldown/layout state alongside existing user state.

**Tech Stack:** Python 3, Tkinter/ttk, Telethon, JSON/SQLite-style existing state layer, unittest, GitHub Actions patch/overlay release pipeline.

**Spec:** `docs/superpowers/specs/2026-08-31-v433-unified-monitoring-design.md`

## Global Constraints

- Base release is published `4.3.2`; target is `4.3.3`.
- Preserve existing user data under `%LOCALAPPDATA%\TelegramAdminManager\data`.
- Respect Telegram FloodWait; do not implement rate-limit evasion.
- Preserve X Planner, updater, forum/topics, recovery, >2 GB Premium routing, and >4 GB skip behavior.

---

### Task 1: Live caption propagation

**Files:** `batch_uploader.py`, `tests/test_v433_unified_monitor.py`

- [x] Add failing tests proving rule edits update waiting queue snapshots, active batch metadata, and live task metadata.
- [x] Verify RED.
- [x] Update rule-edit propagation and resolve caption immediately before each `send_file()`.
- [x] Verify a currently-started send retains its old caption while the next unsent file receives the edited caption.

### Task 2: Unified monitoring workspace

**Files:** `batch_uploader.py`, `telegram_admin_manager.py`, `tests/test_v433_unified_monitor.py`

- [x] Add failing tests that monitoring no longer exposes separate Queue/Rules tabs.
- [x] Build one rules/status table with queue count, current file, progress and speed.
- [x] Keep durable queue internal for restart recovery.
- [x] Add current/recent-files lower pane and retain manual `Upload now` action.

### Task 3: Persistent resizable logs/layout

**Files:** `telegram_admin_manager.py`, `batch_uploader.py`, `tests/test_v433_flood_log_rights.py`

- [x] Add failing tests for permanent right-side log panes and saved sash positions.
- [x] Use resizable paned windows for tool/log layout and monitoring vertical split.
- [x] Persist and restore sash positions with safe bounds.
- [x] Verify all three primary tool windows open with the live log on the right.

### Task 4: FloodWait cooldown and maximum admin rights

**Files:** `telegram_admin_manager.py`, `tests/test_v433_flood_log_rights.py`, `tests/test_incremental_admins_v432.py`

- [x] Add failing tests for persisted cooldown deadlines and local skip before Telegram calls.
- [x] Store FloodWait cooldown in state and continue independent work without sleeping through the wait.
- [x] Make capability rights maximal and schema-adaptive; leave anonymous separate.
- [x] Preserve Telegram helper methods with regression coverage.

### Task 5: Release verification

**Files:** `UPDATE_4_3_3.txt`, release overlay, `release.json`

- [x] Run compile and full unittest suite locally: 70/70 passing.
- [x] Run GUI smoke for all primary tools, right-side logs, unified monitoring and resizable panes.
- [x] Rebuild from clean published 4.3.2 and rerun tests.
- [ ] Run GitHub Actions PR candidate build and verify exact candidate artifact.
- [ ] Merge only after green candidate CI.
- [ ] Verify published v4.3.3 asset and `update.json` SHA/version.
