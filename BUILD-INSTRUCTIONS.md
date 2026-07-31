# Rebuilding gnome-shell with the screen-shield memory-leak patch

Patch: `screenShield-reuse-unlock-dialog-to-fix-memory-leak.patch`
Target: `gnome-shell 50.1-0ubuntu1.1` (matches your extracted source tree)

## What the patch does

- `_completeDeactivate()` no longer calls `this._dialog.destroy()` /
  `this._dialog = null`. It calls `this._dialog.popModal(); this._dialog.hide();`
  instead, so the `UnlockDialog` Clutter actor tree survives across lock
  cycles.
- `_ensureUnlockDialog()` is restructured so the `UnlockDialog` constructor
  and the `'failed'` / `'wake-up-screen'` signal connections only run once
  (first lock ever), while `_refreshBackground()` and `dialog.open()` (which
  `show()`s the actor and re-acquires the modal grab) run on *every* call,
  since those need to happen each time the dialog is re-shown.
- The dialog's own `AuthPrompt` child (the actual password-verification
  widget) is unaffected — `unlockDialog.js` already lazily creates/destroys
  it via `_ensureAuthPrompt()` / `_maybeDestroyAuthPrompt()` on every
  clock↔prompt transition, so no stale auth state persists across cycles.

Verified: patch applies cleanly with `patch -p1` against the pristine
`js/ui/screenShield.js` in this tree, and the resulting file passes
`node --check`.

## 1. Install build dependencies

```bash
sudo apt-get build-dep gnome-shell
sudo apt-get install devscripts quilt fakeroot
```

## 2. Get a clean source tree (or reuse the one you already have)

If you still have the extracted tree from `apt-get source gnome-shell=50.1-0ubuntu1.1`,
just `cd` into it. Otherwise:

```bash
mkdir ~/gnome-shell-build && cd ~/gnome-shell-build
apt-get source gnome-shell=50.1-0ubuntu1.1
cd gnome-shell-50.1
```

## 3. Drop in the patch

```bash
cp /path/to/screenShield-reuse-unlock-dialog-to-fix-memory-leak.patch debian/patches/
echo "screenShield-reuse-unlock-dialog-to-fix-memory-leak.patch" >> debian/patches/series
```

(If you're using the tree from the uploaded tarball, this is already done —
the patch is in `debian/patches/` and appended to `debian/patches/series`.)

## 4. Bump the changelog (recommended, keeps apt/dpkg from getting confused later)

```bash
dch -l~localfix "Reuse unlock dialog in screenShield.js to fix per-cycle memory leak"
```

This produces a version like `50.1-0ubuntu1.1~localfix1`, which sorts *above*
the stock package so it won't be silently reverted by a routine
`apt upgrade`, but is obviously a local build if you ever need to tell it
apart.

## 5. Build

```bash
dpkg-buildpackage -us -uc -b
```

This builds in the parent directory. Expect it to produce (among others):

```
../gnome-shell_50.1-0ubuntu1.1~localfix1_amd64.deb
```

A full rebuild of gnome-shell typically takes several minutes to ~20+
depending on your machine — it links against mutter/GJS/Clutter but doesn't
need to rebuild those.

## 6. Install and test

```bash
sudo apt install ../gnome-shell_50.1-0ubuntu1.1~localfix1_amd64.deb
```

Then log out and back in (screen-shield code only loads at shell startup —
`Alt+F2 r` does *not* reliably reload it under Wayland; a full session
restart is the only sure way).

### Verifying the fix

Same methodology as your original investigation:

```bash
# baseline
grep Private_Dirty /proc/$(pidof gnome-shell)/smaps_rollup

# lock (Super+L), wait a beat, unlock, repeat ~10x

# after N cycles
grep Private_Dirty /proc/$(pidof gnome-shell)/smaps_rollup
```

Private_Dirty growth per cycle should now be near-zero instead of the
~4 MB/cycle you measured before. Also sanity-check:

- Password prompt still appears and unlocks correctly.
- Wrong-password / cancel / user-switch flows still work (these exercise
  `_onUnlockFailed`, `popModal`, and `_ensureAuthPrompt`'s reset path).
- Lock → suspend → resume → unlock still works (exercises `wake-up-screen`).
- Multi-monitor: unplug/replug an external monitor while locked, confirm
  the background still refreshes correctly on next unlock attempt (this is
  what the `_refreshBackground()` call outside the `if (!this._dialog)`
  block is specifically for).

## 7. Rollback

```bash
sudo apt install --reinstall gnome-shell=50.1-0ubuntu1.1.1
```

(or whatever the exact stock version is — check `apt list --installed | grep gnome-shell` beforehand if unsure, or just keep the stock `.deb` from `apt-get source`'s build-dep step / `apt-cache` around before installing the patched one.)

## Note on holding the package

Since this is a local rebuild, a routine `apt upgrade` that pulls in a newer
stock `gnome-shell` will replace it (version `~localfix1` only protects
against the *same* upstream version, not future ones). If you want to pin it:

```bash
sudo apt-mark hold gnome-shell
```

Remember to `apt-mark unhold gnome-shell` and re-diff against the new
upstream `screenShield.js` next time Ubuntu ships a gnome-shell update, since
line numbers/context will likely have shifted.
