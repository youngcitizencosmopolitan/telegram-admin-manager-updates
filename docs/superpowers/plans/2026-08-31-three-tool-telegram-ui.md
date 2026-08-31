# Three-Tool Telegram UI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the Telegram dashboard with a simple launcher for three independent tools while preserving the existing Telegram engines, persistent state, updater, and X Planner.

**Architecture:** Keep Telegram/domain logic in the existing classes and repositories. Change the root Tk UI into a lightweight launcher that opens existing workflows in dedicated `Toplevel` windows, then add reusable numeric tree sorting for live upload speed. Do not migrate or reset user data.

**Tech Stack:** Python 3, Tkinter/ttk, Telethon, SQLite state store, unittest, GitHub Actions patch-based release pipeline.

**Spec:** `docs/superpowers/specs/2026-08-31-three-tool-telegram-ui-design.md`

## Global Constraints

- Base release is exactly `4.3.0`; target release is `4.3.1`.
- Preserve `%LOCALAPPDATA%\\TelegramAdminManager\\data` compatibility.
- Preserve existing Telegram FloodWait/safety handling; do not add limit bypassing.
- Preserve X Planner and updater access.
- Do not rewrite working Telegram creation/upload/admin engines unless required for window isolation.

---

### Task 1: Obtain and verify the exact 4.3.0 source

**Files:**
- Create temporarily on branch: `release-src/4.3.1/00-bootstrap.patch`
- Modify temporarily: `release.json`

**Interfaces:**
- Consumes: existing release pipeline rebuilding from `base_version`.
- Produces: a CI candidate ZIP containing the exact 4.3.0 codebase plus only an `APP_VERSION` bump.

- [ ] **Step 1: Add a bootstrap patch that only changes `APP_VERSION` from `4.3.0` to `4.3.1`.**
- [ ] **Step 2: Point branch `release.json` at base `4.3.0` and the bootstrap patch.**
- [ ] **Step 3: Open a PR to trigger the existing candidate workflow.**
- [ ] **Step 4: Verify CI passes, download the candidate artifact, and use it as the exact local source baseline.**

### Task 2: Write failing tests for launcher and numeric speed sorting

**Files:**
- Modify: `tests/test_ui_helpers.py` or create it if absent.
- Modify as needed: `tests/test_group_factory.py` only if existing admin-selection helpers need coverage.

**Interfaces:**
- Produces: tests for `parse_transfer_speed(text) -> float`, sort direction behavior, and root launcher routing helpers.

- [ ] **Step 1: Add tests asserting `12 MB/s > 900 KB/s > 800 B/s`, em dash/empty values become zero, and comma decimal input is accepted.**
- [ ] **Step 2: Add a test asserting the first click on the Speed heading requests descending order.**
- [ ] **Step 3: Add lightweight tests for the launcher action mapping: groups, monitoring, admins.**
- [ ] **Step 4: Run the focused tests and verify they fail before implementation.**

### Task 3: Replace dashboard root UI with the three-tool launcher

**Files:**
- Modify: `telegram_admin_manager.py`

**Interfaces:**
- Consumes: existing methods/windows for group factory, batch uploader/monitoring, admin assignment, accounts/settings, updater, logs and X Planner.
- Produces: `_build_launcher_ui()`, `_open_group_creation_tool()`, `_open_monitoring_tool()`, `_open_admin_tool()` and reusable tool-window lifecycle helpers.

- [ ] **Step 1: Change the root UI build path so dashboard cards/metric mirrors are not created.**
- [ ] **Step 2: Build three large primary buttons: `Создание групп`, `Мониторинг`, `Админы`.**
- [ ] **Step 3: Add secondary controls/menu for accounts/settings, logs/diagnostics, updater and X Planner.**
- [ ] **Step 4: Reuse one `Toplevel` instance per tool; closing hides/destroys only the presentation window and never duplicates background workers.**
- [ ] **Step 5: Remove dashboard refresh dependencies from periodic UI refresh so missing dashboard widgets cannot raise errors.**
- [ ] **Step 6: Run launcher/helper tests and compile the application.**

### Task 4: Make mass group creation match the simplified workflow

**Files:**
- Modify: `telegram_admin_manager.py` and/or the existing group-factory UI section.
- Modify only if necessary: `tam_core/group_factory.py`.

**Interfaces:**
- Consumes: saved accounts, existing group creation plan, existing granular admin-rights engine.
- Produces: a dedicated group-creation window that creates groups and optionally promotes selected saved accounts immediately.

- [ ] **Step 1: Keep creator account selection, group count and title template as the primary inputs.**
- [ ] **Step 2: Keep saved-account multi-selection for members/admins and the existing rights preset.**
- [ ] **Step 3: Hide unrelated optional publishing/avatar/description controls from the normal creation path.**
- [ ] **Step 4: Ensure result rows distinguish creation success from partial admin-assignment failure.**
- [ ] **Step 5: Run group-factory and Telegram helper unit tests.**

### Task 5: Make monitoring an independent working window and add sorting

**Files:**
- Modify: `batch_uploader.py`
- Modify: `telegram_admin_manager.py`
- Test: `tests/test_ui_helpers.py`

**Interfaces:**
- Produces: `parse_transfer_speed(text) -> float` and reusable Treeview sorting callbacks that preserve selected sort across refreshes.

- [ ] **Step 1: Add a pure transfer-speed parser supporting B/s, KB/s, MB/s and GB/s.**
- [ ] **Step 2: Add heading callbacks for File, Task, Account, Size, %, Speed and Status; Speed defaults to descending on first click.**
- [ ] **Step 3: Keep sort state when live rows are refreshed/mirrored.**
- [ ] **Step 4: Ensure the monitoring window exposes rules, queue, activity, start/stop/continue and error/log access without requiring the old dashboard.**
- [ ] **Step 5: Run monitoring/core-state tests and focused sorting tests.**

### Task 6: Make admin assignment an independent working window

**Files:**
- Modify: `telegram_admin_manager.py`

**Interfaces:**
- Consumes: saved accounts, target groups/channels, existing controller selection and rights engine.
- Produces: a dedicated admin window with batch results.

- [ ] **Step 1: Open the existing admin-management workspace directly from the launcher.**
- [ ] **Step 2: Keep saved user accounts as the only admin candidates in the simplified flow.**
- [ ] **Step 3: Keep multi-target selection, rights preset and per-account/per-target results.**
- [ ] **Step 4: Verify FloodWait/errors remain surfaced in results/logs.**

### Task 7: Build release patch and verify full candidate

**Files:**
- Replace: `release-src/4.3.1/00-bootstrap.patch` with final patch files.
- Modify: `release.json`
- Optionally create: `release-src/4.3.1/UPDATE_4_3_1.txt` via patch/overlay.

**Interfaces:**
- Produces: verified `TelegramAdminManager_v4.3.1_update.zip` candidate.

- [ ] **Step 1: Generate unified patches from the exact 4.3.0 baseline to the implemented 4.3.1 tree.**
- [ ] **Step 2: Run `python -m compileall` and the complete unittest suite locally.**
- [ ] **Step 3: Update branch `release.json` with final patches and release notes.**
- [ ] **Step 4: Let PR CI rebuild from official 4.3.0 and verify all tests.**
- [ ] **Step 5: Download the CI candidate and verify version/files.**
- [ ] **Step 6: Merge only after green CI; the existing push workflow publishes the GitHub release and writes `update.json` last.**
