# Building Foresight in a Fedora toolbox

Minimal package list to run `npm run build`,
`npm run install-extension`, and `npm run lint` inside a fresh
`fedora-toolbox` container. Does **not** cover the nested GNOME Shell
session — run that on the host instead.

## Create the toolbox

``` bash
toolbox create -c foresight-dev
toolbox enter foresight-dev
```

## Install packages

``` bash
sudo dnf install -y gnome-shell nodejs npm
```

| Package | Why |
|----|----|
| `gnome-shell` | Provides the `gnome-extensions` CLI used by `npm run build` and `npm run install-extension`. |
| `nodejs` | Runtime for `eslint` (used by `npm run lint`). |
| `npm` | Installs dev dependencies declared in `package.json` and runs the project’s scripts. |

`git` and `dbus-tools` (for `dbus-run-session`) are already in the
default toolbox image.

## Verify

``` bash
npm install              # installs eslint + eslint-plugin-jsdoc into node_modules/
npm run lint             # should exit clean
npm run build            # produces foresight@pesader.dev.shell-extension.zip
npm run install-extension  # writes to ~/.local/share/gnome-shell/extensions/ (host-visible)
```

After `npm run install-extension`, log out of the host GNOME session and
back in to load the new extension. The toolbox shares `$HOME`, so the
host GNOME Shell picks up the install automatically.
