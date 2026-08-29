# Noctalia / niri Tweaks

**Date:** 2026-08-25
**System:** CachyOS, niri 26.04 + Noctalia 5.0.0-beta.9
**Config layout:** `~/.config/niri/config.kdl` includes split files from `~/.config/niri/cfg/` (`keybinds.kdl`, `layout.kdl`, `rules.kdl`, ...)

## Square windows instead of rounded

### Symptom

All windows render with rounded corners.

### Cause

In niri 26.04, corner rounding is **not** a border/focus-ring option — it is done with window rules:

```kdl
// ~/.config/niri/cfg/rules.kdl
window-rule {
    geometry-corner-radius 20   // rounds every window to 20px
    clip-to-geometry true       // actually clips the window content to the rounded shape
}
```

Note: `border { radius }` / `focus-ring { radius }` do **not exist** in niri 26.04 — putting them in `layout {}` fails config validation with:

```
Error: × unexpected node `radius`
```

### Fix

Edit `~/.config/niri/cfg/rules.kdl`:

```kdl
window-rule {
    geometry-corner-radius 0 // Square windows
    clip-to-geometry false
}
```

### Verify

```bash
niri validate    # must print "config is valid"; niri hot-reloads on save
```

## Related niri tweaks (documented separately)

- Global Super+C / Super+V copy-paste: see `global-copy-paste.md`
  - `Mod+C`/`Mod+V` bound in `cfg/keybinds.kdl` → `~/.local/bin/global-copy-paste.sh`
  - center-column moved from `Mod+C` to `Mod+Shift+C`
