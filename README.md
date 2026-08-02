# GNOME Shell 50.1 — Memory Leak Fixes

Three memory leaks fixed in the screen blank/lock/unlock cycle. Combined
these stop ~1-10 MB RSS growth per cycle on GNOME Shell 50.1 (Ubuntu 26.04).

## Leak 1: Unlock Dialog Destroy/Recreate (4-10 MB/cycle)

**File:** `screenShield.js`, `unlockDialog.js`

Every lock/unlock destroys and rebuilds the entire UnlockDialog widget tree
(password entry, clock, notifications, background actors with GPU textures).
GJS doesn't GC between cycles, so the old actor tree piles up in anonymous
heap.

**Fix:** Reuse the dialog — hide on deactivate, re-show on activate.
Patch: `screenShield-reuse-unlock-dialog-to-fix-memory-leak.patch`

## Leak 2: Lightbox Fade Animation (~1 MB/cycle)

**File:** `screenShield.js`

The idle-timeout fade-to-black uses a full-screen St.Widget (Lightbox) with
`background-color: black`. Each show/hide cycle allocates Cairo surfaces and
Cogl textures that are never freed.

**Fix:** Skip the lightbox fade entirely, go directly to lock screen activation.
Patch hunk in `0002-Fix-screen-blank-leaks-add-DPMS-auto-lock.patch`

## Leak 3: Lock Screen Wallpaper Texture (~1 MB/cycle)

**File:** `screenShield.js`

The lock screen `_lockDialogGroup` has a CSS `background-image: url(...)`
that St loads as a texture. The texture is retained after `actor.hide()`.

**Fix:** Clear the CSS style on deactivate (`set_style(null)`), restore via
`_refreshBackground()` on re-activate.
Patch hunk in `0002-Fix-screen-blank-leaks-add-DPMS-auto-lock.patch`
Related upstream: https://gitlab.gnome.org/GNOME/gnome-shell/-/issues/9188

## Feature: DPMS Auto-Lock

**File:** `screenShield.js`

Connects to Mutter's DisplayConfig `PowerSaveMode` property. When the
display enters DPMS power-save (hardware blank), the screen auto-locks.
When it wakes, the unlock dialog is shown. This bypasses the gnome-shell
idle animation path entirely for screen blanking.

Configuration after install:
```bash
gsettings set org.gnome.desktop.session idle-delay 0
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-timeout 600
```

## Building & Installing

See BUILD-INSTRUCTIONS.md. TL;DR:
```bash
apt source gnome-shell
cd gnome-shell-50.1
# Apply patches
dpkg-buildpackage -us -uc -b
sudo dpkg -i ../gnome-shell*.deb ../gnome-shell-common*.deb
```

Alt+F2, `r` to restart shell.
