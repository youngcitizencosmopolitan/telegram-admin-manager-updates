# Telegram Admin Manager 0001 — Monitoring Pipeline and Serial Version Design

## Scope

Build release **0001** on top of published 4.3.8. Two changes ship together:

1. Monitoring upload packages may overlap so idle saved accounts can begin the next waiting monitoring rule while the previous rule still has in-flight files.
2. User-facing versions switch from semantic-style `4.3.x` to four-digit serial releases: `0001`, `0002`, `0003`, ...

Existing upload safety behavior remains unchanged: one Telethon client per session, one upload at a time per session, global parallel limits, Premium routing, persistent FloodWait, persistent `Ограничен Telegram`, FilePartMissingError tail requeue, captions, ETA, pause/stop and upload history.

## Monitoring Pipeline

The queue dispatcher may combine consecutive waiting **monitoring-origin** queue items into one pipeline wave. Manual/quick-upload packages keep their existing behavior and are not silently merged with monitoring items.

Queue order remains the priority order. A later monitoring task does not begin merely because it exists in the same wave. Each task prepares its file queues and waits for the preceding task's queued files to be assigned to workers. Once the preceding task has no unassigned queued file left, the next task may start competing only for account locks/global slots that become free. This permits overlap with the preceding task's final in-flight uploads without stealing already-queued work from it.

Example: VIP 8 has one file already assigned to account A. Its local queues are empty, so VIP 7 is released. Accounts B/C/D can start VIP 7 immediately while A finishes VIP 8.

### Safety and ordering

- A Telegram session still has exactly one long-lived client in the wave.
- `account_locks` continue to enforce one active upload per account.
- `global_sem` continues to enforce the configured global parallel limit.
- Premium files remain routed only to Premium accounts.
- Later tasks never duplicate a file already claimed by an earlier task.
- Consecutive monitoring queue items with overlapping task IDs are not coalesced into the same wave; this avoids collapsing two snapshots of the same rule.
- If a non-monitoring queue item is encountered, pipeline collection stops so global queue order is preserved.
- On Stop, all unfinished original queue items from the wave are reinserted in original order; upload history prevents already-completed files from being resent.
- Per-queue-item durable status and monitoring signatures are finalized independently even though execution used one shared wave.

## UI status

Multiple monitoring queue rows/rules may show `Выполняется` simultaneously. The queue status summary should report the number of active pipeline items rather than always `Выполняется 1`.

## Version scheme

The application-visible version becomes exactly `0001`. Future visible versions increment numerically as four digits.

Because existing 4.3.8 clients compare manifest versions numerically by dot-separated parts, the update manifest keeps an internal compatibility version while exposing the serial version separately:

- release/tag/package/app version: `0001`
- manifest compatibility `version`: `4.3.8.0001`
- manifest `display_version`: `0001`

The 0001 updater must understand `display_version`:

- legacy current versions such as `4.3.8` compare against manifest compatibility `version`, allowing upgrade to 0001;
- serial current versions such as `0001` compare against `display_version`, preventing an update loop;
- when an update exists, UI/download/install status uses `display_version`, not the compatibility string.

The release workflow must support optional `manifest_version` / `display_version` fields in `release.json`, while release tags/assets continue using the visible `version` field.

## Testing

Add regression tests for:

1. pipeline collection takes only consecutive monitoring items and stops at manual items;
2. overlapping task IDs are not merged into one wave;
3. later task release occurs when the previous task has no unassigned queued files, while account/global locks remain authoritative;
4. queue status can represent multiple active pipeline items;
5. Stop/requeue preserves original item order;
6. updater offers 0001 to 4.3.8 from manifest `4.3.8.0001` + `display_version=0001`;
7. updater does not offer the same manifest to already-installed 0001;
8. updater offers `0002` to 0001 when display version increments;
9. release candidate rebuilds cleanly from published 4.3.8 and all existing tests remain green.
