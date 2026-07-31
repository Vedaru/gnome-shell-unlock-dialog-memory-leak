# GNOME Shell 50.1 - Unlock Dialog Memory Leak Fix

## Problem

Every time you lock and unlock your screen (Super+L → unlock), GNOME Shell
leaks 4-10 MB of memory. This happens because `_completeDeactivate()` in
`js/ui/screenShield.js` calls `this._dialog.destroy()` and
`this._dialog = null`, throwing away the entire UnlockDialog Widget tree
(password entry, clock, notifications, background actors with GPU-backed
Cogl textures). On the next lock, `_ensureUnlockDialog()` builds a brand
new one.

GJS (SpiderMonkey) does not run garbage collection between lock/unlock
cycles, so the destroyed actor tree's heap memory accumulates as anonymous
heap (Private_Dirty) across every cycle. On a typical 30 GB system this
isn't catastrophic, but the memory grows linearly — the more you lock,
the more you leak.

## Fix

**ScreenShield.js:** Instead of destroying the dialog on deactivation,
hide it and reuse it. `_ensureUnlockDialog()` creates the dialog only
once; subsequent locks just re-show it with `_refreshBackground()` and
`dialog.open()`.

**UnlockDialog.js:** A new `resetToClock()` method snaps the dialog
back to the clock page and destroys the `AuthPrompt` before hiding,
so the next lock cycle always shows the clock first instead of a stale
password prompt.

## Files

| File | What changed |
|------|--------------|
| `screenShield-reuse-unlock-dialog-to-fix-memory-leak.patch` | DEP-3 patch against gnome-shell 50.1-0ubuntu1.1 |
| `BUILD-INSTRUCTIONS.md` | How to rebuild gnome-shell with this patch |
| `screenShield.js` | Full patched file (for reference) |
| `unlockDialog.js` | Full patched file (for reference) |

## Apply

```bash
apt-get source gnome-shell=50.1-0ubuntu1.1
cd gnome-shell-50.1
cp screenShield-reuse-unlock-dialog-to-fix-memory-leak.patch debian/patches/
echo "screenShield-reuse-unlock-dialog-to-fix-memory-leak.patch" >> debian/patches/series
dpkg-buildpackage -us -uc -b
sudo dpkg -i ../gnome-shell_*.deb
```

Then log out and back in.

## Verify

```bash
# Before lock/unlock
grep Private_Dirty /proc/$(pidof gnome-shell)/smaps_rollup

# Lock/unlock 10 times

# After: should be near-zero growth instead of +4-10 MB per cycle
grep Private_Dirty /proc/$(pidof gnome-shell)/smaps_rollup
```

## License

Same as gnome-shell (GPL-2.0+).
