# Fixing Light/Dark Mode Not Propagating to GTK3 Apps

**Date:** 2026-08-24
**System:** CachyOS (niri + Noctalia 5.0.0-beta.9)

## Problem

Toggling light/dark mode in Noctalia settings didn't affect other apps (GTK3 apps stayed dark).

## Cause

Noctalia only propagates the freedesktop color-scheme setting:

```bash
gsettings get org.gnome.desktop.interface color-scheme   # prefer-light / prefer-dark
```

- **GTK4/libadwaita and Qt6 apps** follow `color-scheme` (via xdg-desktop-portal) — these update fine
- **GTK3 apps** ignore `color-scheme` and use the legacy `gtk-theme` key instead, which Noctalia never touches

On this system `gtk-theme` was stuck at `adw-gtk3-dark` regardless of mode.

## Solution

A small watcher script mirrors `color-scheme` → `gtk-theme` on every change.

### 1. Script: `~/.local/bin/gtk3-color-scheme-sync.sh`

```bash
#!/bin/bash
# Mirror org.gnome.desktop.interface color-scheme to gtk-theme
# so GTK3 apps follow Noctalia's light/dark toggle.

set_scheme() {
    case "$(gsettings get org.gnome.desktop.interface color-scheme)" in
        *dark*) theme="adw-gtk3-dark" ;;
        *)      theme="adw-gtk3" ;;
    esac
    current=$(gsettings get org.gnome.desktop.interface gtk-theme | tr -d "'")
    [ "$current" != "$theme" ] && gsettings set org.gnome.desktop.interface gtk-theme "$theme"
}

set_scheme
gsettings monitor org.gnome.desktop.interface color-scheme | while read -r _; do
    set_scheme
done
```

### 2. Service: `~/.config/systemd/user/gtk3-color-scheme-sync.service`

```ini
[Unit]
Description=Sync gtk-theme with color-scheme (GTK3 apps)
After=graphical-session.target

[Service]
Type=simple
ExecStart=%h/.local/bin/gtk3-color-scheme-sync.sh
Restart=on-failure

[Install]
WantedBy=graphical-session.target
```

### 3. Install

```bash
chmod +x ~/.local/bin/gtk3-color-scheme-sync.sh
systemctl --user daemon-reload
systemctl --user enable --now gtk3-color-scheme-sync.service
```

## Verification

```bash
noctalia msg theme-mode-set dark
gsettings get org.gnome.desktop.interface gtk-theme   # -> 'adw-gtk3-dark'

noctalia msg theme-mode-set light
gsettings get org.gnome.desktop.interface gtk-theme   # -> 'adw-gtk3'
```

Both keys now switch together; GTK3 apps pick up the new theme live.

## Notes

- Requires the `adw-gtk3` / `adw-gtk3-dark` themes (already installed via pacman)
- Apps that only read the theme at startup need a restart to reflect the change
- Rollback: `systemctl --user disable --now gtk3-color-scheme-sync.service`
