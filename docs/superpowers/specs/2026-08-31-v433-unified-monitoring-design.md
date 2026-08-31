# Telegram Admin Manager 4.3.3 — Unified Monitoring Design

## Goal

Make monitoring a single operational workspace instead of separate queue/rules tabs, apply edited captions to all files that have not started sending yet, keep a live log permanently on the right, persist panel sizes, and make admin assignment resilient to Telegram FloodWait without waiting in the UI.

## Monitoring model

A monitoring rule is the user-facing object. The durable upload queue remains internal in `state.db` for crash/restart recovery. The main monitoring table shows group, folder, caption, pending/new files, status, current file, progress, and speed. A lower resizable pane shows current/recent files.

Editing a rule caption updates the persisted rule, waiting queue snapshots, the active batch snapshot, and live task metadata. The caption is resolved immediately before each `send_file()` call. A file whose Telegram send has already begun keeps the old caption; every later not-yet-started file uses the new caption.

## Layout

Each primary Telegram tool uses a horizontal resizable paned layout: tool content on the left and the shared live log on the right. Monitoring additionally has a vertical resizable split between the rule table and current/recent files. Sash positions are persisted and restored with safe bounds.

## FloodWait

FloodWait is a Telegram server-side limit and is not bypassed. When an admin account receives FloodWait, store an absolute cooldown deadline in `state.db`, skip that account locally until the deadline expires, and continue independent work where possible. Do not sleep through the wait in the UI and do not lose already completed cached assignments.

## Admin rights

Owned accounts are always promoted with the maximum capability rights exposed by the installed MTProto/Telethon `ChatAdminRights` schema. `anonymous` remains a separate visibility setting. Welcome-message management is included automatically when the current schema exposes its corresponding admin-right field; if the installed schema does not expose it, the log reports that limitation rather than pretending the right was sent.

## Compatibility and safety

Preserve `%LOCALAPPDATA%\TelegramAdminManager\data`, X Planner, updater behavior, upload recovery, forum/topic behavior, >2 GB Premium routing and >4 GB skip behavior. Respect Telegram FloodWait and do not add evasion/rotation logic.

## Verification

Release must rebuild from published 4.3.2, compile, pass the complete unit suite, pass GUI smoke for all three tools and right-side logs, and pass GitHub Actions candidate verification before merge.
