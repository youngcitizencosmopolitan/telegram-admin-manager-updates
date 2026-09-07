# Adaptive Telegram Scheduler updates

This directory is the update channel for the Windows launcher.

- `release.json` is the input for the release workflow.
- `update.json` is generated only after a Windows build and GitHub Release succeed.
- `release-src/<version>/source.zip.b64.*` stores the verified source bundle in text chunks so GitHub Actions can rebuild the Windows executable.

The app checks `update.json`, downloads the release EXE over HTTPS, verifies SHA-256, replaces itself through a temporary updater, waits for a healthy restart, and rolls back automatically if the new build does not start successfully.

Application data under `%APPDATA%\\AdaptiveTelegramScheduler` is not part of the release asset and is preserved across updates.
