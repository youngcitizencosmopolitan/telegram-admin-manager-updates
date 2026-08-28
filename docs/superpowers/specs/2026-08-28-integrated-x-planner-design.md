# Telegram Admin Manager 4.3.0 — Integrated X Planner Design

## Goal
Add a `Twitter / X` section to Telegram Admin Manager so the user can manage many X accounts, assign videos to a specific or automatically selected account, generate a future calendar from configurable time windows, remember what is already scheduled, and open the correct signed-in browser profile for the final native scheduling action on x.com.

## Product boundary
This feature does **not** use the X API, X Pro, Selenium, Playwright, browser extensions, DOM scripting, or any other automation that clicks or fills x.com. The application automates only local planning, account/profile selection, calendar generation, caption preparation, file selection assistance, and browser-profile launching. The final upload/date/time/Schedule interaction remains the user's normal action on x.com.

## Application integration
The installed product remains **Telegram Admin Manager** and continues to use the existing update channel and installer.

The main Telegram dashboard gets a prominent `Twitter / X` button. Clicking it launches the same packaged executable with a private command-line mode, for example:

`TelegramAdminManager.exe --x-planner`

The X planner runs in a separate process. This gives the user one installed application while isolating X UI/database/video scanning from Telegram's Telethon workers and dashboard responsiveness.

When running from Python source, the launcher uses the same entry point with the same `--x-planner` argument.

## X accounts
An X account record contains:
- display handle, e.g. `@account1`;
- enabled/disabled state;
- browser type: Chrome or Edge;
- browser executable path or automatic detection;
- browser user-data directory when custom;
- browser profile directory, e.g. `Default`, `Profile 2`;
- optional color/name label for the local UI only;
- scheduling weight for weighted-random selection, default `1`.

The application never stores the X password, cookies, OAuth tokens, or session secrets. Login remains inside the normal browser profile.

The account editor has `Open profile` and `Test profile` actions. These only launch the configured profile to `https://x.com/home`.

## Content jobs
A content job contains:
- video path;
- video fingerprint based on normalized path, file size, and modification time;
- user caption;
- optional reusable global suffix;
- account selection mode: `specific`, `random`, `round_robin`, or `least_loaded`;
- specific account id when applicable;
- assigned account id after planning;
- planned local date/time;
- status: `unplanned`, `planned`, `scheduled`, `skipped`, `published_or_expired`;
- notes;
- created/updated timestamps.

The final text shown to the user is `caption + blank line + suffix` when both are non-empty.

The same video fingerprint cannot be silently added twice. The UI asks whether to keep a deliberate duplicate.

## Time windows
The planner supports any number of configurable local time windows. A window contains:
- enabled state;
- start time, e.g. `09:00`;
- end time, e.g. `12:00`;
- applicable weekdays;
- capacity per account per day, default `1`;
- randomize time inside the window, default enabled.

Global scheduling settings contain:
- timezone: system local time;
- start planning date;
- maximum posts per account per day, default `4`;
- minimum gap between posts for the same account, default `120` minutes;
- random minute granularity, default `1` minute;
- optional inter-account offset range, default disabled;
- planning horizon, default `90` days.

The typical default configuration is four windows per day:
- 09:00–12:00
- 13:00–16:00
- 17:00–20:00
- 21:00–23:30

## Assignment modes
### Specific
The job is always assigned to the chosen enabled account.

### Random
Choose randomly from enabled accounts that still have capacity for the target date/window. Weighted random uses each account's scheduling weight.

### Round robin
Rotate deterministically through enabled accounts, continuing from the previous assignment cursor.

### Least loaded
Choose the enabled account with the fewest planned/scheduled jobs in the current planning horizon; ties are resolved randomly.

## Planning algorithm
For each unplanned job in queue order:
1. Determine candidate accounts from the job's selection mode.
2. Scan dates from the configured start date forward up to the planning horizon.
3. For each date, scan enabled windows in chronological order.
4. Reject account/window combinations that exceed window capacity, daily account capacity, or minimum-gap rules.
5. If randomization is enabled, draw an available minute within the window that respects the rules; otherwise use the window start or first valid minute.
6. Persist the chosen account and timestamp immediately.
7. If no candidate exists within the horizon, leave the job `unplanned` and show a clear reason.

Replanning can operate on selected jobs or all `planned` jobs. Jobs already marked `scheduled` are immutable unless the user explicitly resets them to `planned`.

## Main X UI
The X planner opens with four functional areas:

### Accounts
Compact list of enabled accounts showing:
- handle;
- browser profile;
- planned today / daily limit;
- next planned time;
- enable toggle.

Actions: `Add`, `Edit`, `Open profile`.

### Queue
Table columns:
- status;
- planned date/time;
- account;
- video filename;
- caption preview;
- selection mode.

Actions:
- `Add videos`;
- `Edit caption`;
- `Set account mode`;
- `Plan selected`;
- `Plan all`;
- `Clear plan`;
- `Mark scheduled`;
- `Skip`.

Multiple videos can be imported in one operation. The account mode can be applied to the entire selection.

### Calendar
Views: `Today`, `Tomorrow`, `7 days`, `30 days`, and per-account filter.

Each item shows exact planned time, account, filename, and scheduling state. Scheduled items are visually distinct from merely planned items.

### Next task
A focused workflow for manual native scheduling on x.com:
- exact account;
- exact date and time;
- video path;
- final caption;
- `Open X profile`;
- `Copy caption`;
- `Open video folder`;
- `Mark scheduled & next`.

`Open X profile` launches the configured Chrome/Edge profile directly to `https://x.com/compose/post`; it does not interact with the page after launch.

## Data storage
Use a dedicated SQLite database under the existing application data root, named `x_planner.sqlite3`.

The database is separate from Telegram's existing state database. X planner schema migrations are independent and cannot modify Telegram tables.

Tables:
- `x_accounts`
- `x_time_windows`
- `x_settings`
- `x_jobs`
- `x_planner_state`

All writes use short transactions. No periodic polling of video folders is required in v1, keeping idle CPU usage close to zero.

## Browser launching
Chrome and Edge are detected from common Windows install locations. The user may override the executable path.

Chrome launch shape:
`chrome.exe --profile-directory="Profile 2" https://x.com/compose/post`

Edge launch shape:
`msedge.exe --profile-directory="Profile 2" https://x.com/compose/post`

If a custom user-data directory is configured, add `--user-data-dir=<path>`.

The application does not launch browsers with remote-debugging flags and does not inspect browser processes/pages.

## Clipboard and file assistance
`Copy caption` uses the application's own Tk clipboard API. It is always a user-triggered action.

`Open video folder` opens Windows Explorer and selects the file where supported. The application does not automatically attach the file to x.com.

## Performance
- X planner is lazy: no X module/process starts until the user clicks `Twitter / X`.
- Separate process prevents X UI work from blocking Telegram's Tk event loop.
- SQLite queries use indexes on `planned_at`, `assigned_account_id`, and `status`.
- Calendar queries are range-limited; no unbounded table scans on every refresh.
- Large videos are never decoded or transcoded by this feature.
- Video fingerprinting uses metadata only, not full-file hashing.

## Error handling
- Missing video: job remains visible with `missing file` warning.
- Missing browser executable/profile: block `Open X profile` and show repair action.
- No available schedule slot: job stays `unplanned` with reason.
- Database failure: show error and avoid silently changing status.
- Browser launch failure: do not mark job scheduled.

No X password/session data is written to logs.

## Release/update strategy
Target version: **Telegram Admin Manager 4.3.0**.

The existing Telegram update manifest remains the single update channel. The release package includes the new X planner module and tests. The standalone `X Post Manager 1.0.0` release may remain available, but Telegram Admin Manager 4.3.0 does not depend on it and does not launch it.

## Tests
Pure/unit tests cover:
- caption + suffix composition;
- time-window validation;
- minimum-gap enforcement;
- specific/random/round-robin/least-loaded assignment;
- daily/window capacity;
- planning horizon exhaustion;
- scheduled jobs not being silently replanned;
- duplicate video fingerprint detection;
- Chrome/Edge launch command construction;
- UI view-model formatting;
- command-line `--x-planner` dispatch.

Release CI must run the existing Telegram tests plus the new X planner tests before publishing 4.3.0.

## Explicit exclusions for 4.3.0
- X API integration.
- X Pro integration.
- Selenium/Playwright/browser extension automation.
- Automatic clicking, typing, uploading, or scheduling on x.com.
- Password/cookie/token storage.
- Automatic media conversion or splitting.
- Cloud scheduler while the PC is off.
