# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working
with code in this repository.

## Project

Foresight is a GNOME Shell Extension (GJS / ES modules) that opens the
activities overview automatically when the current workspace becomes
empty, or when the user switches to an already-empty workspace. It
targets GNOME Shell 46–50 — changes that touch shell APIs must remain
compatible across that range (see `foresight@pesader.dev/metadata.json`
→ `shell-version`).

## Commands

``` bash
npm install                # installs eslint@9.19.0 + eslint-plugin-jsdoc from package.json
npm run build              # gnome-extensions pack → foresight@pesader.dev.shell-extension.zip
npm run install-extension  # build, then install via gnome-extensions install --force
npm run lint               # eslint "**/*.js" --no-warn-ignored
npm run lint:fix           # same, with --fix
npm run clean              # remove the .zip and docs/

# Nested gnome-shell sessions for manual testing (not npm scripts; just raw commands):
env MUTTER_DEBUG_DUMMY_MODE_SPECS=1256x768 dbus-run-session -- gnome-shell --nested --wayland
env MUTTER_DEBUG_NUM_DUMMY_MONITORS=2 dbus-run-session -- gnome-shell --nested --wayland
```

Note: `npm run install` will NOT work — npm intercepts the `install`
argument and runs the install lifecycle instead of the script. Always
use `npm run install-extension`.

The lint config (`eslint.config.mjs`) composes `lint/eslintrc-gjs.mjs`
(upstream GJS rules) and `lint/eslintrc-shell.mjs` (Shell-specific
rules). After enabling the extension in the nested session, reload it
with `Alt+F2` → `r` is **not** available on Wayland — re-run
`npm run install-extension` and restart the nested session.

## Architecture

All extension logic lives in a single file:
`foresight@pesader.dev/extension.js`. The exported `Extension` subclass
just constructs/destroys one `Foresight` instance on
`enable()`/`disable()`.

The design is **event-driven**, not polling. `Foresight` connects to
three signals:

- `workspace-switched` on `global.workspace_manager` — rebinds the
  per-workspace listener to the new active workspace, then either hides
  or shows the overview based on window count.
- `window-removed` on the **current** workspace — re-connected every
  time the active workspace changes (see `_connectWorkspaceSignals` /
  `_disconnectWorkspaceSignals`). This is the only correct way to track
  removals on whatever workspace the user is actually on.
- `hidden` on `Main.overview` — resets `_activatedByExtension` so we
  never auto-hide an overview the user opened themselves.

The `_activatedByExtension` flag is the critical invariant: **Foresight
only ever hides the overview if it opened it.** Preserve this when
modifying show/hide logic.

### Why the show-activities path sleeps

`_windowRemoved` waits `_getWindowCloseAnimationTime(window)` ms before
showing the overview, so the window-close animation finishes first. The
timing constants (`DESTROY_WINDOW_ANIMATION_TIME`,
`DIALOG_DESTROY_WINDOW_ANIMATION_TIME`) are copied from gnome-shell’s
`windowManager.js` and must stay in sync with it. If animations are
disabled in `St.Settings`, the delay is 0.

### Window filtering

Two filters decide whether a window “counts”:

- `_windowAccepted` — drops hidden windows, anything that isn’t
  NORMAL/DIALOG/MODAL_DIALOG, and (when
  `org.gnome.mutter workspaces-only-on-primary` is true) windows on
  secondary monitors.
- `_isTemporaryWindow` — hard-coded allowlist of splash/updater/progress
  windows (LibreOffice startup, DBeaver progress, Steam launcher,
  Discord updater, …) that we never want to treat as “the last real
  window.” Add to the `temporaryWindows` array (matching `title` +
  `wmClass` + `sandboxedAppId`) when users report regressions caused by
  app-specific transient windows. The LibreOffice case uses a regex
  because its title includes the version number.

### Cleanup

`destroy()` disconnects every signal, cancels the pending `_sleep`
timeout if any, and nulls fields. Any new signal connections or timers
must be torn down here — leaking them on `disable()` will keep callbacks
firing after the extension is gone and break re-enable.
