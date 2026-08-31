# Telegram Admin Manager — three-tool UI design

## Goal
Simplify the Telegram part of Telegram Admin Manager into three independent working tools instead of a single dashboard: **Mass group creation**, **Monitoring**, and **Admin assignment**.

## Product structure

The root window becomes a small launcher. It contains three primary actions:

1. **Создание групп** — open the existing group/channel factory workflow in a dedicated window.
2. **Мониторинг** — open the existing batch uploader/monitoring workflow in a dedicated window.
3. **Админы** — open the existing account/group admin assignment workflow in a dedicated window.

Secondary functions (accounts, API/settings, logs, updater, diagnostics, X Planner) remain available through compact secondary controls/menu, but are not mixed into the Telegram workflow.

## Mass group creation

This is an independent window. The essential workflow is:

- choose one or more creator/owner accounts;
- choose how many groups to create;
- choose the title/template already supported by the factory;
- choose which saved user accounts should be added to each new group and promoted to administrators;
- start creation;
- show per-group result: created, admins assigned, partial result, or error.

No new avatar/description/content publishing workflow is added. Existing Telegram safety/FloodWait handling remains intact.

## Monitoring

Monitoring stays independent from group creation and admin assignment. The window keeps:

- folder-to-group rules;
- background scanning;
- queue state and recovery;
- new-file detection;
- live file/activity table;
- numeric upload speed;
- errors/log access;
- manual start/stop/continue controls.

Tables that show live uploads must support persistent user sorting, especially **Speed descending/ascending**. Sorting by speed is numeric (B/s, KB/s, MB/s, GB/s), not lexical.

## Admin assignment

Admin assignment is a separate window for already existing groups/channels:

- choose saved user accounts;
- choose target groups/channels;
- apply the existing rights preset/rights engine;
- assign in batch;
- show per-target/per-account result and Telegram errors;
- preserve existing controller/owner selection logic and FloodWait handling.

Only the user's saved Telegram accounts are used as admin candidates; no contact scraping or arbitrary-user discovery is introduced.

## Shared state

The three tools share the existing persistent account store, state database, created-groups registry, monitor rules, upload history, Telegram sessions and logs. Closing one tool must not destroy background queue/monitor state. Existing data in `%LOCALAPPDATA%\\TelegramAdminManager\\data` must remain compatible.

## UI behavior

- No dashboard cards, mirrored queue tables, metric tiles, or dashboard sash persistence in the root window.
- Each tool opens in its own reusable `Toplevel` window and can be closed/reopened without duplicating background workers.
- Root window remains usable while a tool window is open.
- Existing advanced UI may remain instantiated internally when needed for compatibility, but it is not the normal user-facing workspace.
- X Planner remains available as a secondary, independent feature and is not deleted.

## Versioning

Base release: 4.3.0. Target release: 4.3.1.

## Acceptance criteria

- Launching the app shows a simple launcher instead of the dashboard.
- The three primary Telegram tools open independently and continue to use existing data/state.
- Mass creation can immediately add selected saved accounts as admins using existing Telegram logic.
- Monitoring still detects and queues new files while uploads are running.
- Upload/activity tables can be sorted by speed numerically; first click on Speed sorts fastest first.
- Existing unit tests pass and new UI/helper tests cover launcher routing and numeric speed sorting.
- X Planner and updater remain available and functional.
