# CLAUDE.md — ncdu

Fork of the official ncdu disk usage browser (https://dev.yorhel.nl/ncdu), maintained at g.blicky.net/ncdu.git. Written in Zig (0.16+). Links ncursesw and zstd.

## Build

```bash
zig build                        # output in zig-out/bin/ncdu
zig build -Doptimize=ReleaseSafe # optimized
```

Requires: `zig`, `libncursesw-dev`, `libzstd-dev`

## Source layout

| File | Purpose |
|------|---------|
| `src/main.zig` | Entry point, arg parsing, config, main event loop, state machine |
| `src/browser.zig` | Interactive file browser: rendering, key handling, sort/nav |
| `src/ui.zig` | ncurses helpers, box drawing, `runCmd`, `spawnDetached` |
| `src/model.zig` | In-memory tree: Entry, Dir, Link, Ext |
| `src/scan.zig` | Filesystem scan |
| `src/delete.zig` | Delete UI and logic |
| `src/c.zig` | C interop: ncursesw, zstd, unistd, etc. |
| `src/util.zig` | String/number formatting utilities |

## State machine (`main.zig:119`)

```
state: enum { scan, browse, refresh, shell, delete }
```

`browser.zig` drives `.browse`. Key handlers either mutate state directly or transition to another state (e.g. `main.state = .shell`). The `shell` state tears down curses, runs a child, re-inits. Background spawning (e.g. `xdg-open`) uses `ui.spawnDetached` and does NOT go through the state machine.

## Key handler pattern (`browser.zig:935`)

All keypresses go through `browser.keyInput(ch: i32)`. Add new bindings to the main `switch (ch)` block. Also add to the `help.keys` array (`browser.zig:675`) — it's a flat `[][:0]const u8` of alternating `"key", "description"` pairs. `keylines = 10` controls how many display at once; extras are paged.

## Spawning background processes (`ui.spawnDetached`)

Use for anything that should run without blocking or tearing down the ncurses UI (e.g. `xdg-open`). Double-forks so the grandchild is orphaned by init — no zombies. Redirects stdio to `/dev/null`. Calls `execvp` via C bindings.

Do NOT use `ui.runCmd` for background processes — it tears down and re-inits curses (correct for interactive shells, wrong for GUI launchers).

## Config flags (`main.zig:100`)

Boolean capability flags (`can_delete`, `can_shell`, `can_open`, etc.) are `?bool` — null means "use default". Defaults are resolved after arg parsing (around line 583). Add `--enable-X` / `--disable-X` pairs to `argConfig()`.

## C interop

`src/c.zig` cimports: `stdio.h`, `string.h`, `time.h`, `wchar.h`, `locale.h`, `fnmatch.h`, `unistd.h`, `sys/types.h`, `pwd.h`, `curses.h`, `zstd.h`, and on Linux `sys/vfs.h`. Access via `const c = @import("c.zig").c`.

For posix primitives not in c.zig, prefer `std.posix` (fork, waitpid, open, dup2, close, exit).

## Local patches

- **`'o'` key — open with xdg-open**: `browser.zig` builds the full path and calls `ui.spawnDetached`. Disabled in binreader mode (no real paths). Controlled by `config.can_open`.
