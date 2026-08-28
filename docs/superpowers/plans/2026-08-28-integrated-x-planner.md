# Integrated X Planner Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship Telegram Admin Manager 4.3.0 with a `Twitter / X` button that launches a separate-process local X scheduling planner for multi-account video calendars without X API or website automation.

**Architecture:** Keep Telegram's existing runtime unchanged except for command-line dispatch and a launcher button. The X planner lives in focused modules, uses its own SQLite database, and is launched as the same packaged executable with `--x-planner`, isolating its Tk loop and data access from Telegram workers. The planner automates local assignment/calendar generation only; x.com interaction remains manual.

**Tech Stack:** Python 3, Tkinter/ttk, sqlite3, pathlib, subprocess, webbrowser/browser executables, unittest, existing Telegram Admin Manager packaging/update pipeline.

**Spec:** `docs/superpowers/specs/2026-08-28-integrated-x-planner-design.md`

## Global Constraints
- Target release version is `4.3.0` with base version `4.2.9`.
- One installed Telegram Admin Manager and one existing update channel.
- No X API, X Pro dependency, Selenium, Playwright, DOM scripting, remote debugging, password/cookie/token storage, or automated x.com interaction.
- X planner must not start until the user clicks `Twitter / X` or launches `--x-planner`.
- X planner stores data in a dedicated `x_planner.sqlite3` database under the existing application data root.
- Existing Telegram tests must remain green.
- New X planner tests must be green before publishing.

---

### Task 1: Recover and verify the exact 4.2.9 release source

**Files:**
- Create temporarily: `.github/workflows/export-telegram-source.yml`
- Produce artifact: `telegram-admin-manager-4.2.9-source.zip`

**Interfaces:**
- Consumes: public release asset `TelegramAdminManager_v4.2.9_update.zip`.
- Produces: exact extracted 4.2.9 source tree used as the patch base.

- [ ] Add a manual/PR workflow that downloads the published v4.2.9 update ZIP from the repository release.
- [ ] Verify the ZIP SHA-256 equals `af13a5328b8487d0eaf5ef80345e50c453bc4372179dd2199a122cb8b135346e`.
- [ ] Extract it and upload the source tree as a GitHub Actions artifact.
- [ ] Download the artifact through the GitHub connector and inspect actual entrypoint, package layout, tests, and packaging files.
- [ ] Remove the temporary export workflow before final merge unless it proves generally useful.

### Task 2: X planner domain model and SQLite storage

**Files:**
- Create in the recovered source tree: `x_planner.py` or focused `x_planner/` modules following the source layout discovered in Task 1.
- Create tests in the existing test layout.

**Interfaces:**
- Produces `compose_x_post_text(caption: str, suffix: str) -> str`.
- Produces `video_fingerprint(path: Path) -> str` based on normalized path, size, and mtime.
- Produces `XPlannerStore` methods for accounts, time windows, settings, jobs, status transitions, and planner cursor.

- [ ] Write failing tests for caption composition, duplicate fingerprints, schema creation, persistence across reopen, and scheduled-job immutability.
- [ ] Run the focused tests and verify they fail because production code is missing.
- [ ] Implement schema with tables `x_accounts`, `x_time_windows`, `x_settings`, `x_jobs`, `x_planner_state` and indexes on planned time/account/status.
- [ ] Implement repository methods with short transactions and no changes to Telegram DB/schema.
- [ ] Re-run focused tests to green.

### Task 3: Time-window validation and assignment planner

**Files:**
- Create/modify the X planner scheduling module.
- Add focused tests.

**Interfaces:**
- Produces `validate_time_window(start: str, end: str) -> tuple[int, int]` minute offsets.
- Produces `plan_jobs(store, job_ids, start_date, horizon_days, rng) -> PlanningResult`.
- Supports account modes `specific`, `random`, `round_robin`, `least_loaded`.

- [ ] Write failing tests for four default windows, per-window capacity, max posts/day, minimum gap, and planning-horizon exhaustion.
- [ ] Write failing deterministic tests for all four account assignment modes using seeded `random.Random`.
- [ ] Implement candidate slot generation and constraints.
- [ ] Implement weighted random, persisted round-robin cursor, and least-loaded tie randomization.
- [ ] Prove `scheduled` jobs are excluded from automatic replanning unless reset explicitly.
- [ ] Run scheduling tests to green.

### Task 4: Browser profile command builder and local helper actions

**Files:**
- Create/modify X planner browser helper module.
- Add focused tests.

**Interfaces:**
- Produces `detect_browser(kind: str) -> Path | None`.
- Produces `build_browser_command(account, url='https://x.com/compose/post') -> list[str]`.
- Produces user-triggered helpers to open X profile, copy caption, and reveal video in Explorer.

- [ ] Write failing command-construction tests for Chrome/Edge default profile, named profile, and custom user-data directory.
- [ ] Implement common Windows browser-path detection and explicit path override.
- [ ] Implement browser launch without remote-debugging or DOM hooks.
- [ ] Implement Tk clipboard copy and Explorer file selection as explicit user actions.
- [ ] Run helper tests to green.

### Task 5: X planner UI

**Files:**
- Create/modify X planner UI module(s).
- Add non-GUI view-model tests.

**Interfaces:**
- Consumes `XPlannerStore`, planning service, and browser helper.
- Produces a standalone Tk root for `--x-planner` mode.

- [ ] Write failing tests for queue-row formatting, account load summaries, next-task selection, and calendar range filtering.
- [ ] Build lazy X planner window with Accounts, Queue, Calendar, and Next task areas.
- [ ] Add dialogs for account/browser profile editing, global suffix, daily limits, time windows, and planning settings.
- [ ] Add bulk video import, per-selection account mode changes, caption editing, `Plan selected`, `Plan all`, `Clear plan`, `Mark scheduled`, and `Skip`.
- [ ] Add views `Today`, `Tomorrow`, `7 days`, `30 days`, plus account filter.
- [ ] Ensure missing video/browser errors stay visible and never mark a job scheduled automatically.

### Task 6: Integrate launcher into Telegram Admin Manager

**Files:**
- Modify actual 4.2.9 entrypoint discovered in Task 1.
- Modify main dashboard UI discovered in Task 1.
- Add dispatch tests.

**Interfaces:**
- `--x-planner` starts only the X planner root and does not initialize Telethon/Telegram dashboard workers.
- Main dashboard `Twitter / X` action launches the same packaged executable in a child process.

- [ ] Write failing dispatch test proving `--x-planner` selects X mode.
- [ ] Add command-line mode selection before Telegram app initialization.
- [ ] Add `Twitter / X` button in the main dashboard without restructuring unrelated Telegram UI.
- [ ] Implement source-mode and frozen-PyInstaller child launch paths.
- [ ] Prove clicking/opening X does not import/start X planner during normal Telegram startup until requested.

### Task 7: Release patch, regression tests, and CI

**Files:**
- Create: `release-src/4.3.0/01-x-planner.patch` (or split focused patches if required by current workflow).
- Create/modify: `release-src/4.3.0/*-tests.patch`.
- Modify: `release.json` to version `4.3.0`, base `4.2.9`.
- Reuse existing `.github/workflows/publish-release.yml` unless a minimal packaging inclusion fix is necessary.

**Interfaces:**
- Existing update manifest remains `update.json` and is published by the current workflow only after a successful build/test.

- [ ] Generate patch(es) from exact 4.2.9 source to 4.3.0 source.
- [ ] Update version constant to `4.3.0` and release notes describing the integrated X planner.
- [ ] Run all existing Telegram tests plus all new X planner tests on Windows CI.
- [ ] Build the exact update package using the existing pipeline.
- [ ] Verify resulting update ZIP contains X planner code and normal Telegram entrypoint.
- [ ] Verify existing Telegram release/update workflow behavior is otherwise unchanged.

### Task 8: Final release verification

**Files:**
- No new production files unless verification exposes a defect.

**Interfaces:**
- Produces published `TelegramAdminManager_v4.3.0_update.zip` and updated `update.json`.

- [ ] Verify CI conclusion is success after fresh run.
- [ ] Verify release tag `v4.3.0` exists and contains the expected update ZIP.
- [ ] Verify `update.json.version == 4.3.0` and SHA-256 matches the release asset.
- [ ] Verify release notes are correct.
- [ ] Confirm no standalone X API credentials or browser-automation dependencies were introduced.
