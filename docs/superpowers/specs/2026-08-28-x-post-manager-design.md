# X Post Manager v1 — Design

## Goal
Build a standalone Windows desktop application that manages multiple X accounts, queues different videos, lets the user write captions manually with an optional reusable suffix, schedules four or more posts per account per day, automatically prepares oversized videos, and publishes through the official X API.

## Product boundary
X Post Manager is a separate application from Telegram Admin Manager. Both live in the same GitHub repository but use independent source trees, versions, databases, release manifests, binaries, and GitHub Actions workflows. Telegram 4.2.9 must not be modified by this feature.

## Compliance boundary
The app must not automate the X website with Playwright/Selenium/browser scripting. X explicitly prohibits non-API automation. Posting and media upload use the official X API with OAuth 2.0 Authorization Code + PKCE.

Required OAuth scopes: `tweet.read tweet.write users.read media.write offline.access`.

## Windows application
- App name: `X Post Manager`
- Initial version: `1.0.0`
- Target: Windows 10/11 x64
- Runtime: Python 3.12, packaged with PyInstaller in one-directory mode
- UI: Tkinter/ttk to keep the runtime light and avoid a browser UI
- Local data: SQLite under `%APPDATA%/XPostManager/xpostmanager.db`
- Sensitive account tokens: encrypted with Windows DPAPI before storage
- Logs: `%APPDATA%/XPostManager/logs/app.log`

## Main UI
One dashboard window with three primary areas:
1. Accounts panel: connected account, status, enable/disable, add/remove.
2. Queue table: scheduled time, account, filename, caption preview, status, progress/error.
3. Controls: add videos, edit caption, auto-distribute, schedule settings, start/pause scheduler, retry failed.

Dialogs are allowed for account connection, settings, caption editing, and auto-distribution rules.

## Captions
Each queue item stores a user-written caption. The app also stores one reusable global suffix. Final post text is:

`caption + blank line + suffix`

when both are non-empty. The UI must preview the exact final text and show the current character count. The app does not generate marketing text automatically in v1.

## Scheduling and distribution
The user can configure daily time slots, with four slots per day as the expected common case. Auto-distribution assigns distinct queued videos across enabled accounts round-robin and fills future slots in chronological order. The user can manually reassign an item or change its time after distribution.

The scheduler runs while the app is running (including minimized-to-tray mode), checks due jobs frequently, and persists all state so a restart does not lose the queue. Missed jobs become due immediately after restart unless the user has paused scheduling.

## Video preparation
Use bundled `ffmpeg.exe` and `ffprobe.exe`.

Default safe limits for a standard X post are:
- target segment duration: 135 seconds
- hard duration limit: 140 seconds
- target maximum file size: 500 MB
- hard file size limit: 512 MB

Workflow:
1. Probe duration, codec, dimensions, and size.
2. If within limits and MP4/H.264/AAC-compatible, use original.
3. If duration is too long, split into numbered parts at approximately 135 seconds.
4. If a resulting part exceeds 500 MB or is incompatible, transcode that part to H.264/AAC MP4.
5. Each generated part becomes its own queue item and inherits the original caption, with an optional `Part N/M` suffix controlled by a setting.

Prepared media is stored under `%APPDATA%/XPostManager/prepared/` and can be cleaned after successful publication.

## X API integration
OAuth uses a local callback listener on `127.0.0.1` and opens the user's default browser only for the authorization screen. The application stores a Client ID configured by the user; no X credentials are hardcoded in the repository.

Media upload uses the X API v2 chunked upload flow: initialize, append chunks, finalize, poll processing status. Publishing uses `POST /2/tweets` with the uploaded media ID and final caption.

Refresh tokens are used automatically when access tokens expire. Account identity is confirmed using the authenticated-user endpoint after connection.

## Reliability
- SQLite transaction before and after each state transition.
- Jobs have statuses: `queued`, `preparing`, `ready`, `uploading`, `publishing`, `published`, `failed`, `paused`.
- Network/API failures use bounded retries with exponential backoff.
- A process crash must not mark an unpublished item as published.
- Logs include account handle, queue item id, stage, HTTP status, and a safe error message, but never access or refresh tokens.

## Updater and releases
X Post Manager has its own manifest at `x-post-manager/update.json` and release metadata at `x-post-manager/release.json`.

A dedicated GitHub Actions workflow builds:
- `XPostManager_v1.0.0_update.zip`
- `XPostManager_Setup_v1.0.0.exe`

Release tag format: `x-v1.0.0`.

The application checks the raw `x-post-manager/update.json`, shows available version and release notes, downloads the ZIP, verifies SHA-256, launches a separately packaged updater executable, exits, then the updater replaces application files and restarts the app.

## Tests
Unit tests cover caption composition, scheduling/distribution, database state transitions, token refresh decisions, video segmentation planning, update-manifest parsing, and X API response handling using a fake HTTP transport. Tests must not require live X credentials.

## v1 exclusions
- Browser automation of x.com.
- Automatic AI caption generation.
- Engagement automation (likes, follows, replies, reposts).
- Cross-account duplicate-content posting logic.
- Cloud server operation while the user's PC is off.
