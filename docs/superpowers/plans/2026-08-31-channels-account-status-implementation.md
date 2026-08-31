# Channels Inventory and Restricted Account Status Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship Telegram Admin Manager 4.3.8 with a cached Channels inventory workspace and persistent Telegram anti-spam restriction status for saved accounts.

**Architecture:** Extend the existing `state.db` with additive channel/account-relation/restriction tables. Keep Telegram read inventory separate from risky write actions; Channels opens from cache and refreshes explicitly. Integrate restriction checks at existing admin/join write boundaries instead of creating account rotation or bypass logic.

**Tech Stack:** Python 3, Tkinter/ttk, SQLite, Telethon 1.44.0, unittest, existing GitHub release-patch pipeline.

**Spec:** `docs/superpowers/specs/2026-08-31-channels-account-status-design.md`

## Global Constraints

- Base release is 4.3.7; target release is 4.3.8.
- Preserve existing FilePartMissingError tail-requeue behavior, FloodWait persistence, monitoring ETA, captions, admin-right semantics, sessions and user data.
- FloodWait is not an anti-spam restriction.
- Do not implement Telegram limit evasion, account rotation to bypass restrictions, or high-frequency polling.
- Channels tab must read cache immediately and perform network reads only on explicit refresh/single-peer refresh.

---

### Task 1: Persistent storage for inventory and restrictions

**Files:**
- Modify: `tam_core/state_db.py`
- Create: `tests/test_v438_state_inventory.py`

**Interfaces:**
- Produces: `channel_upsert(payload)`, `channels_all()`, `channel_account_put(peer_id, session, payload)`, `channel_accounts(peer_id)`, `account_restriction_get(session)`, `account_restriction_set(session, payload)`, `account_restriction_clear(session)`.

- [ ] Write failing tests that create a temporary `StateDatabase`, verify schema version increment, channel upsert/merge, per-account relation storage and persistent restriction set/clear.
- [ ] Run the focused test and confirm failure because the tables/methods do not exist.
- [ ] Add `channels`, `channel_accounts`, `account_restrictions` tables additively and implement the methods above using short-lived existing DB sessions.
- [ ] Run focused tests and the existing state-db tests.

### Task 2: Restriction classifier and risky-action gate

**Files:**
- Create: `tam_core/account_restrictions.py`
- Modify: `tam_core/__init__.py`
- Modify: `telegram_admin_manager.py`
- Create: `tests/test_v438_account_restrictions.py`

**Interfaces:**
- Produces: `classify_telegram_restriction(exc) -> str | None`, `is_risky_membership_write(operation) -> bool`, app helpers `account_restriction(session)`, `remember_account_restriction(account, exc, operation)`, `account_is_restricted(session)`.

- [ ] Write failing tests: `PeerFloodError`/`UserRestrictedError` classify restricted; FloodWait, privacy, FilePartMissing, ChatAdminRequired and network errors do not.
- [ ] Run focused test and confirm RED.
- [ ] Implement conservative class-name/message classifier; no live probing or bypass behavior.
- [ ] At `_promote_one_target_async` selected-account join phase and `_factory_join_admin`, skip persistently restricted accounts with explicit journal message before any risky Telegram write.
- [ ] On matching anti-spam errors at those boundaries, persist restriction reason/source/time.
- [ ] Keep FloodWait path unchanged and independent.
- [ ] Run focused and existing admin/FloodWait tests.

### Task 3: Channel inventory model and scanner

**Files:**
- Create: `tam_core/channel_inventory.py`
- Modify: `telegram_admin_manager.py`
- Create: `tests/test_v438_channel_inventory.py`

**Interfaces:**
- Produces: `serialize_admin_rights(rights) -> dict[str,bool]`, `relation_from_entity(entity) -> dict`, `merge_channel_observation(existing, observation) -> dict`, app async methods `_refresh_channels_inventory_async()` and `_refresh_channel_async(peer_id)`.

- [ ] Write failing tests for peer-ID dedupe, owner/admin/member mapping and dynamic serialization of exposed boolean admin-right fields.
- [ ] Run focused test and confirm RED.
- [ ] Implement pure inventory helpers.
- [ ] Implement explicit full refresh: for each authorized saved account call `get_dialogs()`, accept channels/chats, derive public link from username, merge by canonical peer ID, persist relation and timestamps; one account failure must not abort scan.
- [ ] Treat absence after a completed account scan as `absent` only for peers seen by another saved account in the same refresh; preserve `unknown` when the account scan failed.
- [ ] Implement single-peer refresh using cached peer identity and safe read-only entity lookup, updating only that peer/account relation.
- [ ] Respect persistent FloodWait before read calls and keep partial results.
- [ ] Run focused tests.

### Task 4: Channels UI, views, filters and copy actions

**Files:**
- Modify: `telegram_admin_manager.py`
- Create: `tests/test_v438_channels_ui_contract.py`

**Interfaces:**
- Adds main tab key `channels` between `admins` and `accounts`.
- Produces UI methods `_build_channels_tab()`, `refresh_channels_from_cache()`, `_channels_apply_view()`, `_channels_copy(field)`, `refresh_channels_from_telegram()`, `refresh_selected_channel()`.

- [ ] Write failing source/UI-contract tests for permanent tab order, default columns and persisted visible-column guard that keeps `Название` when all fields are hidden.
- [ ] Run focused test and confirm RED.
- [ ] Add `tab_channels` and build it before Accounts/API.
- [ ] Build toolbar with `Обновить каналы`, `Обновить этот канал`, search, three filters and `Вид` menu.
- [ ] Build cached Treeview columns: `Название`, `Ссылка`, `Тип`, `ID`, `Владелец`, `Наши админы`, `Наши участники`, `Статус`.
- [ ] Persist visible column keys in existing config/state settings and restore on startup; enforce at least `Название` visible.
- [ ] Add selected-peer detail Treeview listing every saved account, role, `может добавлять админов`, rights text, last error/status.
- [ ] Add context menu/double-click actions to copy name/link/ID and open link with `webbrowser.open` when available.
- [ ] Render filters entirely from cache without Telegram calls.
- [ ] Run focused tests and GUI smoke construction.

### Task 5: Accounts UI status and release verification

**Files:**
- Modify: `telegram_admin_manager.py`
- Create: `tests/test_v438_accounts_status.py`
- Add release patch: `release-src/4.3.8/v4.3.8.patch`
- Modify: `release.json`

**Interfaces:**
- Accounts UI visibly reports `OK`, `FloodWait до …`, `⚠ Ограничен Telegram`, or authorization/session problems.
- Restricted-account actions: local `Снять отметку`; conservative `Проверить снова` that uses authorization/read checks only and never join/invite bursts.

- [ ] Write failing tests for status composition and local restriction clearing.
- [ ] Run focused test and confirm RED.
- [ ] Add a status summary Treeview/panel in Accounts/API without breaking the existing account selection list used by current workflows.
- [ ] Implement `Снять отметку` for selected account and warning that this does not remove Telegram-side restrictions.
- [ ] Implement `Проверить снова`: connect, verify authorized session, perform safe `get_me()`/read check, report result; do not auto-clear merely because login works unless no restriction-class error is observed during the conservative check and the user explicitly confirms clearing the local mark.
- [ ] Update `APP_VERSION` to 4.3.8.
- [ ] Run `python -m unittest discover -s tests -v`, `python -m compileall`, and GUI construction smoke.
- [ ] Generate a minimal 4.3.7→4.3.8 release patch and verify it applies to a clean extracted 4.3.7 package.
- [ ] Run the complete test suite on the rebuilt patched package.
- [ ] Publish through PR candidate CI; merge only if candidate rebuild/tests/ZIP succeed; verify final release and `update.json` after merge.
