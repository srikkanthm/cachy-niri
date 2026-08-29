# Global Super+C / Super+V Copy-Paste (macOS-style)

**Date:** 2026-08-24
**System:** CachyOS (niri + Noctalia), terminals: Ghostty + Alacritty

## Problem

Copy/paste was Ctrl+C / Ctrl+V (or Ctrl+Shift+C/V in terminals). Wanted macOS-style **Super+C / Super+V** globally, but niri already used `Mod+C` for `center-column`.

## How it works

Wayland compositors can't inject shortcuts into apps directly. Instead:

1. **niri intercepts Super+C / Super+V** as compositor bindings
2. A script detects the **focused window's app-id** via `niri msg focused-window`
3. It injects the right keystroke with **`wtype`** (virtual keyboard):
   - Normal apps → `Ctrl+C` / `Ctrl+V`
   - Terminals (Ghostty/Alacritty) → `Ctrl+Shift+C` / `Ctrl+Shift+V`, because plain Ctrl+C is SIGINT and Ctrl+V inserts verbatim in shells

## Components

### 1. Dependency

```bash
sudo pacman -S wtype
```

### 2. Script: `~/.local/bin/global-copy-paste.sh`

```bash
#!/bin/bash
# Global Super+C / Super+V -> Ctrl+C / Ctrl+V translation.
# When a terminal is focused, sends Ctrl+Shift+C/V instead,
# because plain Ctrl+C/V are SIGINT / verbatim-insert in shells.
# Usage: global-copy-paste.sh copy|paste

mode="$1"
case "$mode" in
    copy) key="c" ;;
    paste) key="v" ;;
    *) exit 1 ;;
esac

appid=$(niri msg focused-window | awk -F'"' '/App ID/{print $2}')

case "$appid" in
    com.mitchellh.ghostty|Alacritty|org.alacritty)
        wtype -M ctrl -M shift -P "$key" -p "$key" -m shift -m ctrl
        ;;
    *)
        wtype -M ctrl -P "$key" -p "$key" -m ctrl
        ;;
esac
```

### 3. niri bindings (`~/.config/niri/cfg/keybinds.kdl`)

```kdl
// --- Global copy/paste (macOS-style) ---
Mod+C hotkey-overlay-title="Copy (Super+C -> Ctrl+C)" { spawn-sh "~/.local/bin/global-copy-paste.sh copy"; }
Mod+V hotkey-overlay-title="Paste (Super+V -> Ctrl+V)" { spawn-sh "~/.local/bin/global-copy-paste.sh paste"; }
```

### 4. Conflict resolution

niri's center-column moved off Super+C:

```kdl
// was: Mod+C { center-column; }
Mod+Shift+C                         { center-column; }
```

(Also present: `Mod+Ctrl+C` = `center-visible-columns`, unchanged.)

### 5. Terminal direct bindings (optional fallback)

Ghostty (`~/.config/ghostty/config`) also has direct bindings — these are unreachable while niri intercepts Super+C/V first, kept as harmless fallback:

```
keybind = super+c=copy_to_clipboard
keybind = super+v=paste_from_clipboard
```

## Verification

```bash
niri validate                       # config valid; niri hot-reloads on save
which wtype                         # dependency check
# Manual test: focus a GUI app (e.g. Brave/Nautilus),
# select text, Super+C, then Super+V elsewhere
```

## Caveats

- Apps using non-standard copy mechanisms may not respond to synthetic Ctrl+C
- New terminal app-ids must be added to the terminal case in the script
- Rollback: remove the two Mod+C/Mod+V bindings and restore `Mod+C { center-column; }`
