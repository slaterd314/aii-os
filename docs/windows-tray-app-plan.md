# Plan: AII OS as a Windows taskbar app (tray icon + Exit/Restart)

Branch: `feature/windows-tray-app` (created off `main`)

## Correction to the earlier audit

I previously said `cmd/aii-app` was darwin-only. That was wrong — it has
three files: `main.go` (`//go:build darwin`), `main_other.go`, and
`main_windows.go` (`//go:build windows`). The Windows launcher already
exists and already does more than I'd credited it for.

I also want to name plainly what `dan/windows-launch` actually contains,
since my last message assumed it hid the *app's* console window: its two
relevant commits (`223b568`, `47339ae`) touch only
`internal/tools/shell_windows.go` — the subprocess launcher for the
`shell` tool I use. That's the fix that already stopped console windows
popping up when I run shell commands. It never touched how `aii.exe`
itself is launched.

## What already exists today (verified by reading the code, not assumed)

- `AII OS.exe` (built from `cmd/aii-app/main_windows.go`) already launches
  `aii.exe` via `install.StartDetached`, which already sets
  `HideWindow: true` and `CREATE_NO_WINDOW` (`internal/install/detach_windows.go`).
  **`aii.exe` launched this way already shows no console window.** This is
  exactly how I am running right now — I checked my own process tree
  earlier and confirmed it.
- A stop mechanism already exists: `install.Stop(slot)` signals a named
  Win32 event (`internal/install/stop_windows.go`,
  `internal/install/stopname.go`); `aii.exe` listens for it and shuts down.
- `aii.exe` has an internal self-restart path (`internal/app/relaunch.go`,
  exit code `3`, re-execs itself) — but nothing external currently drives it
  except a human stopping and relaunching by hand, which is what the
  operator did manually a moment ago.
- **The actual gap:** `AII OS.exe`'s `run()` starts `aii.exe` detached, waits
  for it to serve, opens the browser tab, and then **returns — the launcher
  process exits**. Nothing stays resident. There is currently no process
  alive whose job is to sit in the taskbar, which means there is nothing to
  hang a tray icon or a context menu off of, regardless of whether
  `aii.exe`'s own console is hidden or not.

## Revised design (operator's simpler framing, adopted)

Do **not** touch `cmd/aii` / `aii.exe` at all. It stays exactly what it is:
a console-subsystem binary that is both the CLI and the server, launched
hidden exactly as today.

Instead, change `cmd/aii-app`'s Windows launcher (`AII OS.exe`) from a
fire-and-exit launcher into the resident taskbar app:

1. **Stay resident.** After starting (or finding already-running) `aii.exe`
   and opening the dashboard once, `main_windows.go` does not exit. It runs
   a Win32 message loop instead.
2. **Tray icon.** Register a `Shell_NotifyIcon` icon for that message loop,
   using an embedded `.ico` resource. Reuse `golang.org/x/sys/windows`
   (already a dependency via `stop_windows.go`) for the raw syscalls —
   `Shell_NotifyIconW`, window class registration, and the message loop —
   rather than pulling in a new tray library, to keep the dependency
   surface unchanged. (If the raw syscall surface turns out to be
   meaningfully large, I'll say so before writing it, and we can weigh a
   small vendored tray package instead — noting the tradeoff rather than
   deciding silently.)
3. **Context menu**, minimum viable:
   - **Open Dashboard** — same `open(url)` already implemented.
   - **Restart** — call the existing `install.Stop(slot)` event, wait for
     the port to stop responding, then call `install.StartDetached` again
     with the same command line already built in `run()`. This is the same
     stop-then-relaunch sequence a human just did by hand, automated.
   - **Exit** — call `install.Stop(slot)` to stop `aii.exe`, remove the
     tray icon, and exit the launcher process.
4. **Startup behavior unchanged.** The existing `-startup` flag (silent,
   no browser open) still applies; it now also means "stay resident with a
   tray icon" rather than "start and exit."
5. **Single instance.** Since the launcher now stays running, add a guard
   (named mutex, Windows-only) so a second `AII OS.exe` launch — e.g. the
   user double-clicking the shortcut again — detects the tray app is
   already running and either focuses/opens the dashboard or exits quietly,
   instead of registering a second tray icon.

## What this deliberately does not change

- `aii.exe`'s subsystem, its CLI behavior, or `cmd/aii/main.go` — untouched.
- The shell-tool console-hiding fix already on `main` — untouched.
- The stop-event protocol and slot/port model — reused as-is, not
  redesigned.

## Open questions for the operator before I write code

1. Is `AII OS.exe` the thing currently pinned to the taskbar / set to run
   at sign-in, or is `aii.exe` launched some other way (scheduled task,
   service, direct shortcut) on this machine? If something launches
   `aii.exe` directly rather than through `AII OS.exe`, that path would
   still show a console window and would need to be repointed at
   `AII OS.exe` for this to actually fix what you see day to day.
2. Icon asset — is there an existing `.ico` for AII OS, or do I need to
   improvise a placeholder?
3. Confirm the menu scope: just Open Dashboard / Restart / Exit, or
   anything else (e.g. "Show logs", identity-slot switching for multi-slot
   installs)?

## Sequencing

1. Get answers to the open questions above (or explicit "proceed anyway").
2. Add the resident message loop + tray icon registration, no menu yet —
   verify a plain icon appears and stays in the taskbar without a console
   window.
3. Add the context menu and wire Open Dashboard.
4. Wire Restart (stop + relaunch) and Exit (stop + quit).
5. Add the single-instance guard.
6. Manual verification pass with the operator watching their actual
   desktop — sandbox-side success is not evidence of what appears on
   their screen.
7. Only then: tests where feasible (the Win32 message loop itself isn't
   meaningfully unit-testable; the stop/restart sequencing logic can be).
