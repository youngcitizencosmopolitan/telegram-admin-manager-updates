# Telegram Admin Manager — Channels Inventory and Restricted Account Status Design

## Scope

Build the next release on top of Telegram Admin Manager 4.3.7. The change adds a persistent **Каналы** workspace for channel/group inventory plus a persistent account restriction status used to avoid repeatedly attempting actions with accounts that Telegram has anti-spam restricted.

This design must not change the existing upload queue, FilePartMissingError requeue behavior, FloodWait cooldown behavior, captions, monitoring ETA, or existing admin-right assignment semantics unless explicitly stated below.

## Goals

1. Give the user one place to inspect all Telegram groups/channels seen by saved accounts.
2. Show which saved accounts are owner/admin/member/not present in each peer and which admin rights they have.
3. Make names, links and IDs easy to copy.
4. Let the user choose which columns are visible and persist that choice.
5. Avoid unnecessary Telegram RPC calls by using a local cache with explicit refresh operations.
6. Persistently mark accounts that Telegram appears to anti-spam restrict and exclude them from membership/admin/mass-action attempts until the user clears or re-checks the status.
7. Keep FloodWait separate from the anti-spam restriction status.

## Main Navigation

Add a permanent top-level tab **Каналы** between **Админы** and **Аккаунты / API**.

The persistent tab order becomes:

`Создание групп | Мониторинг | Админы | Каналы | Аккаунты / API | Twitter / X | Обновления`

The shared journal remains permanently visible on the right exactly as in 4.3.7.

## Channels Table

The main channels table reads from the local cache and has these default columns:

- `Название`
- `Ссылка`
- `Тип`
- `ID`
- `Владелец`
- `Наши админы`
- `Наши участники`
- `Статус`

Rows are deduplicated by canonical Telegram peer identity, primarily peer/channel ID. Different saved accounts seeing the same Telegram channel must contribute to one row rather than duplicate rows.

### Row actions

A selected row supports:

- copy channel/group name;
- copy link when available;
- copy Telegram ID;
- open the public/invite link when available;
- refresh only this peer;
- refresh its per-account membership/admin details.

Double-click should open the most useful action: open the link when available, otherwise select/show details without error.

## Selected Channel Details

Selecting a row shows a detail pane listing every saved Telegram account and its relationship to that peer:

- `Владелец`
- `Админ`
- `Участник`
- `Нет в канале`
- `Не удалось проверить`
- `Ограничен Telegram` when applicable

For admins, show the rights that the installed Telethon schema exposes. At minimum render all current `ChatAdminRights` boolean fields available to the running library, rather than maintaining a permanently hard-coded UI subset.

The detail pane must make it clear which account is the owner and which account can add/promote admins.

## View / Column Configuration

Add a **Вид** control for the Channels table. The user can toggle individual table columns on/off.

At minimum these fields are configurable:

- Название
- Ссылка
- Тип
- ID
- Владелец
- Наши админы
- Наши участники
- Статус

The selection persists across app restarts in user data under `%LOCALAPPDATA%\TelegramAdminManager\data` through the existing persistent state/settings mechanism.

The app must always keep at least one useful identifying column visible; if the user tries to hide all columns, keep `Название` visible.

## Filters

The Channels workspace includes:

- text search by name/link/ID;
- `Только где мы владелец`;
- `Только где есть наши админы`;
- `Только проблемные`.

“Problematic” includes stale/error scan state, unavailable peer, unresolved membership state, or presence of a saved account marked `Ограничен Telegram` for an action the user would normally perform.

## Refresh Model

Use a hybrid cache model, not a full live reload whenever the tab opens.

### Full refresh

`Обновить каналы` scans dialogs/known peers through authorized saved accounts and merges results into the local inventory. It then enriches ownership/admin/member information only as needed.

The scan must:

- respect existing FloodWait persistence/cooldowns;
- skip unauthorized sessions;
- skip accounts currently persistently marked `Ограничен Telegram` for actions that could aggravate anti-spam restrictions, while still allowing safe read-only inventory calls where Telegram permits them;
- avoid duplicate reads of the same peer when cached information is still fresh during the same refresh operation;
- keep partial results if one account fails.

### Single-peer refresh

`Обновить этот канал` refreshes one peer and its per-account memberships/rights without rescanning all dialogs.

### Cache timestamps

Store `last_seen_at`, `last_refreshed_at`, and per-account membership/right refresh timestamps so the UI can show stale/error states without forcing network traffic.

## Persistent Data Model

Extend the existing state database rather than creating a second independent database.

Logical entities:

### Channel inventory

`channels`

- canonical peer ID
- peer type
- title
- username/public link when available
- best known invite/public URL
- last_seen_at
- last_refreshed_at
- scan status/error

### Channel/account relation

`channel_accounts`

- peer ID
- saved account/session key
- relation: owner/admin/member/absent/unknown
- serialized admin rights exposed by current Telethon
- can_add_admins boolean derived from rights/creator state
- last_refreshed_at
- last_error

### Persistent account restriction

`account_restrictions`

- saved account/session key
- status (`restricted` or clear)
- reason category
- raw/friendly last error
- detected_at
- last_seen_at
- source operation

Migrations must be additive and preserve all existing user data, sessions, queue state, monitoring rules, X Planner data and history.

## Telegram Anti-Spam Restriction Status

### Detection

Persist `⚠ Ограничен Telegram` when an operation produces an error strongly associated with Telegram anti-spam/peer-flood restriction, especially `PeerFloodError` and equivalent known RPC errors/messages indicating anti-spam action restriction.

Do **not** mark the account restricted for:

- `FloodWaitError`;
- ordinary network errors;
- `FilePartMissingError`;
- `UserPrivacyRestrictedError` caused by the target user's privacy settings;
- `ChatAdminRequiredError` or missing admin rights;
- temporary media/upload failures.

The restriction record includes the exact source operation and friendly/raw error for diagnostics.

### Behavior while restricted

A persistently restricted account stays visible everywhere but is automatically skipped for actions likely to trigger more anti-spam enforcement, including:

- joining via invite/public link for admin assignment workflows;
- inviting users/channels where that saved account is the actor;
- promotion/admin assignment attempts when the restricted account itself would need to perform the joining/membership step;
- other existing mass membership/admin actions that clearly perform writes through that account.

Read-only inspection may still use it when safe and authorized.

Logs must explicitly say that the account was skipped because it is marked `Ограничен Telegram`, rather than silently omitting it.

### Accounts UI

In **Аккаунты / API**, show a persistent status badge/column:

- `OK`
- `FloodWait до ...` when applicable
- `⚠ Ограничен Telegram`
- authorization/session errors as currently supported

For a restricted account provide:

- `Снять отметку` — clears only the local persistent restriction mark;
- `Проверить снова` — performs a conservative, read-oriented account/session check and clears the mark only when the implementation has positive evidence that the account can safely resume. It must not run a burst of join/invite actions merely to test the restriction.

The UI must explain that clearing the local mark does not remove Telegram-side restrictions.

## Interaction with FloodWait

FloodWait remains an independent persistent cooldown exactly as introduced earlier.

An account can be both:

- persistently restricted by anti-spam status; and
- temporarily under FloodWait.

The UI should surface the more actionable combined state without merging the records internally.

No sleeping through FloodWait and no attempts to bypass Telegram limits are introduced by this feature.

## Error Handling

Channel inventory is best-effort. One failed account or one inaccessible peer must not abort the whole scan.

Persist and display per-peer/per-account errors with bounded logs.

If a link cannot be resolved, retain the channel row and show an empty/unknown link rather than deleting the peer.

If admin rights cannot be fetched, preserve the membership relation and show rights as unavailable/stale.

## Performance and Safety

- The Channels tab opens from cache immediately.
- Network scanning occurs only on explicit refresh or existing controlled background mechanisms added specifically for this feature.
- Do not create high-frequency polling.
- Deduplicate peers across accounts before expensive enrichment where possible.
- Respect existing per-account FloodWait persistence.
- Do not implement limit evasion, account rotation to bypass Telegram restrictions, or automated anti-spam circumvention.

## Testing Requirements

Add tests covering at least:

1. permanent tab order contains `Каналы`;
2. channel rows merge by peer ID across multiple accounts;
3. column visibility persists and cannot hide every identifying column;
4. account/peer roles map to owner/admin/member/absent/unknown correctly;
5. dynamic admin-right serialization includes fields exposed by the current Telethon schema;
6. `PeerFloodError` classifies as a persistent Telegram restriction;
7. FloodWait, privacy errors, FilePartMissing and missing-admin-right errors do not classify as persistent restriction;
8. restricted accounts are skipped for risky write membership/admin actions with an explicit log reason;
9. read-only channel inventory can continue despite one restricted or failed account;
10. `Снять отметку` clears local state only;
11. migrations preserve existing state and tolerate an existing 4.3.7 database;
12. complete existing regression suite still passes;
13. GUI smoke verifies Channels tab, shared right journal and Accounts restriction status render without opening separate tool windows.

## Release Constraints

- Base release: 4.3.7.
- Target release for this feature: 4.3.8 unless another release is published first.
- Preserve `%LOCALAPPDATA%\TelegramAdminManager\data` and all user sessions/history/settings.
- Preserve the classic persistent-tab main-window layout.
- Preserve 4.3.7 FilePartMissingError deferred-tail-requeue behavior.
- Preserve existing X Planner data and behavior.
- Build through the repository release pipeline against the actual published base package before merge.
