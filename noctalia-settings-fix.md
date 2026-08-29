# Fixing Noctalia Settings Not Opening From Launcher

**Date:** 2026-08-24
**System:** CachyOS (niri + Noctalia 5.0.0-beta.9), Vicinae launcher

## Problem

Opening "Noctalia" from the launcher did nothing — the settings window never appeared. Running `noctalia msg settings-open` from a terminal worked fine.

## Cause

The packaged desktop entry (`/usr/share/applications/dev.noctalia.Noctalia.desktop`) runs:

```
Exec=noctalia --daemon
```

When Noctalia is already running, this fails with `error: noctalia is already running` and opens nothing.

The working command only exists in the entry's hidden desktop action:

```
[Desktop Action Settings]
Exec=sh -c "noctalia msg settings-open || { noctalia --daemon && noctalia msg settings-open; }"
```

Launchers like Vicinae execute the main `Exec`, not desktop actions, so the working command was never used.

Note: the IPC socket is keyed per Wayland display (`~/.cache` log showed `/run/user/1000/noctalia-wayland-1.sock`), and both the session and Vicinae's systemd service share `WAYLAND_DISPLAY=wayland-1`, so IPC works fine from the launcher's environment.

## Solution

Created a dedicated desktop entry at `~/.local/share/applications/noctalia-settings.desktop`:

```ini
[Desktop Entry]
Type=Application
Name=Noctalia Settings
Comment=Open Noctalia settings window
Exec=sh -c "noctalia msg settings-open || { noctalia --daemon && noctalia msg settings-open; }"
Icon=noctalia
Terminal=false
Categories=Settings;DesktopSettings;
Keywords=settings;noctalia;
```

The fallback (`noctalia --daemon && ...`) covers the edge case where Noctalia isn't running yet.

## Verification

```bash
# Simulates launching from a launcher
gtk-launch noctalia-settings

# Confirm window exists and is focused
niri msg windows | grep -A3 'Noctalia Settings'
```

## Troubleshooting commands used

- `noctalia msg --help` — list all IPC commands (settings-open/close/toggle, panel-open, etc.)
- `niri msg windows` — list windows (the settings window was confirmed as app-id `dev.noctalia.Noctalia`)
- `tail ~/.cache/noctalia/noctalia.log` — Noctalia debug log
- Check process env: `tr '\0' '\n' < /proc/<pid>/environ | grep -i wayland`
