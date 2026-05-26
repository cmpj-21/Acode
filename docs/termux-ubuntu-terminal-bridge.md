# Termux Ubuntu Terminal Bridge

## Goal

Use the user's existing Termux/proot Ubuntu environment as the backing shell for Acode's terminal, instead of installing and maintaining a separate Alpine Linux environment inside Acode.

This should let Acode reuse the already-configured Ubuntu toolchain, packages, shell config, language runtimes, and project directories from Termux.

## Current Acode Terminal Flow

Acode currently owns its terminal environment:

1. The terminal plugin downloads an Alpine minirootfs into Acode's private app files directory.
2. `Terminal.startAxs()` writes bundled init scripts into that directory.
3. `init-sandbox.sh` starts `proot` with `$PREFIX/alpine` as the root filesystem.
4. `init-alpine.sh` configures Alpine, installs required packages with `apk`, and starts the AXS terminal server.
5. The frontend xterm component connects to AXS over `http://localhost:8767` and `ws://localhost:8767`.

Important files:

- `src/components/terminal/terminal.js`
- `src/components/terminal/terminalManager.js`
- `src/plugins/terminal/www/Terminal.js`
- `src/plugins/terminal/www/Executor.js`
- `src/plugins/terminal/scripts/init-sandbox.sh`
- `src/plugins/terminal/scripts/init-alpine.sh`
- `src/plugins/terminal/src/android/ProcessManager.java`
- `src/plugins/terminal/src/android/TerminalService.java`

## Proposed Direction

Add an external terminal backend mode for Termux Ubuntu.

In that mode:

1. Acode does not install or start its bundled Alpine sandbox.
2. Acode connects its xterm UI to a configured local terminal server address.
3. Termux starts that terminal server inside the existing proot Ubuntu environment.
4. The terminal session therefore runs in the same environment the user already uses from Termux.

The default backend should remain the existing bundled Alpine environment so current users are not broken.

## Implementation Notes

Likely app-side changes:

- Add a terminal backend setting, for example:
  - `Bundled Alpine`
  - `External Termux Ubuntu`
- In external mode, skip `Terminal.isInstalled()` and `Terminal.startAxs()` in `TerminalComponent.createSession()`.
- Let the external mode use a configurable host and port, likely defaulting to `127.0.0.1:8768`.
- Reuse Acode's existing xterm UI and WebSocket attachment path where possible.
- Keep Alpine install, uninstall, backup, and restore behavior scoped to the bundled backend.

Likely Termux-side changes:

- Add a startup script that enters the existing Ubuntu proot distro and starts an AXS-compatible PTY/WebSocket server.
- Prefer using Termux to launch the server because Android app sandboxing prevents Acode from directly owning Termux's private environment.
- Optional later improvement: use Termux's `RUN_COMMAND` intent so Acode can ask Termux to start the bridge automatically.

## Constraints

- Acode and Termux are separate Android apps with separate private app storage and Linux UIDs.
- Acode should not assume it can execute binaries from Termux private storage directly.
- Shared storage is not a reliable place for executable toolchains on Android.
- The Acode frontend expects an AXS-compatible HTTP/WebSocket API, so a plain shell process is not enough.
- Termux `RUN_COMMAND` integration requires user setup:
  - Acode must request `com.termux.permission.RUN_COMMAND`.
  - The user must grant that permission to Acode.
  - Termux must have `allow-external-apps=true` in `~/.termux/termux.properties`.

## Open Questions

- Can the existing `axs` binary run correctly from the user's Termux/proot Ubuntu setup, or do we need a small compatibility server?
- Should external backend configuration live in terminal settings only, or in a broader app settings section?
- Should Acode auto-start the Termux bridge, or should it only connect to a manually started bridge at first?
- What should the reconnect and error UI look like when the external bridge is not running?
- How should paths from the Ubuntu environment map back to Acode file URLs for the `acode` CLI helper?

## First Milestone

Build a manual bridge proof of concept:

1. Start an AXS-compatible server from Termux Ubuntu on a non-default port.
2. Add a temporary external-backend setting in Acode.
3. Confirm Acode can open a terminal tab and attach to that server.
4. Confirm commands run inside the existing Ubuntu environment.
5. Confirm Ctrl+C, resize, shell startup files, and long-running processes behave correctly.
