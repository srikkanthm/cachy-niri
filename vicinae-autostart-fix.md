# Fixing Vicinae Launcher Not Starting After Reboot

**Date:** 2026-08-24
**System:** CachyOS (niri + Noctalia)

## Problem

Vicinae launcher wasn't working after a restart until manually running `vicinae server`.

## Cause

The `vicinae-bin` package ships a systemd **user** service (`/usr/lib/systemd/user/vicinae.service`), but it was **disabled**, so nothing started the server at login.

## Solution

Enable and start the systemd user service:

```bash
systemctl --user enable --now vicinae.service
```

This created the symlink:

```
~/.config/systemd/user/graphical-session.target.wants/vicinae.service
    -> /usr/lib/systemd/user/vicinae.service
```

## What the service does

From `/usr/lib/systemd/user/vicinae.service`:

- Runs `vicinae server --replace` on login (`WantedBy=graphical-session.target`)
- `Restart=always` with `RestartSec=60`, so it recovers automatically if it crashes
- Stops when the graphical session ends (`PartOf=graphical-session.target`)

## Verify

```bash
systemctl --user status vicinae.service
pgrep -af vicinae
```

## Rollback (if ever needed)

```bash
systemctl --user disable --now vicinae.service
```
