# GNOME Shell 50.1 — Memory Leak + DPMS Auto-Lock Fixes

## What this fixes

Three memory leaks eliminated, DPMS-triggered auto-lock added:

| # | Issue | Fix |
|---|---|---|
| 1 | Unlock dialog destroyed each cycle | Reuse dialog (hide/show) |
| 2 | Wallpaper texture retained across lock/unlock | Clear/restore CSS style |
| 3 | Lightbox fade animation | Removed entirely (`_onStatusChanged` is now a no-op) |

## How DPMS auto-lock works

gnome-settings-daemon handles display power (DPMS). When the display blanks,
Mutter's DisplayConfig `PowerSaveMode` property changes, triggering our handler
to call `lock()`. On wake, `_wakeUpScreen()` shows the unlock dialog.

No gnome-shell actors are created during blanking -- zero memory allocation.

## Configuration

```bash
# Idle detection timeout (must be > 0 for g-s-d to work)
gsettings set org.gnome.desktop.session idle-delay 60

# DPMS blank timeout after idle (seconds)
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-timeout 240
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-battery-timeout 240
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'blank'

# Enable locking
gsettings set org.gnome.desktop.screensaver lock-enabled true
gsettings set org.gnome.desktop.screensaver idle-activation-enabled true

# Lock on suspend
gsettings set org.gnome.desktop.screensaver ubuntu-lock-on-suspend true
```

Total time before display blanks + locks: `idle-delay + sleep-inactive-ac-timeout`
(60s + 240s = 5 minutes in this example).

## Building

```bash
apt source gnome-shell
cd gnome-shell-50.1
# Copy the patch into debian/patches/ubuntu/
# Add it to debian/patches/series
dpkg-buildpackage -us -uc -b
sudo dpkg -i ../gnome-shell*.deb ../gnome-shell-common*.deb
```

Alt+F2, `r` to restart shell.

## Upstream

Wallpaper texture leak: https://gitlab.gnome.org/GNOME/gnome-shell/-/issues/9188

## Files

- `screenShield.js` — patched file (all fixes applied)
- `DPMS-auto-lock-and-memory-leak-fixes.patch` — DPMS + wallpaper + fade fixes
- `screenShield-reuse-unlock-dialog-to-fix-memory-leak.patch` — dialog reuse fix
- `unlockDialog.js` — patched unlock dialog (dialog reuse fix)
