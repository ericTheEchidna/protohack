Protohack / nethack-protohack
Meson build system — replace Makefile soup

## Phase N1 — Meson Build

Replace NetHack's traditional `sys/unix/` Makefile system with a Meson
build that produces a working `nethack` binary on Linux, macOS, and
Windows from a single `meson setup builddir && ninja -C builddir`
invocation.

The goal is the simplest possible build first: TTY windowport, no tiles,
no sound, no Qt. Tiles and sound are separate tasks (N2, N3) that layer
on top of this one.

### Context

The repo (`nethack-protohack`) is a clean fork of NetHack `NetHack-3.7`.
It has no CMakeLists.txt and no meson.build anywhere — this is greenfield.

Three git submodules exist and must be handled:
- `submodules/lua` — `github.com/lua/lua` (NetHack embeds Lua for scripting)
- `submodules/pdcurses` — `github.com/wmcbrine/PDCurses`
- `submodules/pdcursesmod` — `github.com/Bill-Gray/PDCursesMod`

On Linux/macOS the system curses is used; pdcurses/pdcursesmod are
Windows-only. Lua is needed on all platforms.

The existing Makefile hints live in `sys/unix/hints/linux.370` and
`sys/unix/hints/macOS.370` — these are the authoritative record of what
source files and defines are needed for each platform. Read them before
writing meson.build.

### Tasks

- [ ] Read `sys/unix/hints/linux.370` and `sys/unix/hints/macOS.370` in
      full — extract the source file lists, CFLAGS, and defines
- [ ] Identify the minimal source set for a TTY build:
  - [ ] `src/` — all `.c` files listed in `Makefile.src` (or hints)
  - [ ] `sys/unix/` — `unixmain.c`, `unixsys.c`, etc.
  - [ ] `win/tty/` — TTY windowport sources
  - [ ] `util/` — `makedefs`, `tilemap`, `dgn_comp`, `lev_comp` build tools
        (these are build-time utilities, not linked into the binary)
- [ ] Handle Lua:
  - [ ] Run `git submodule update --init submodules/lua`
  - [ ] Add `submodules/lua` as a Meson subproject or compile its sources
        directly (Lua is a single-directory C library — direct compilation
        is simpler than a wrap file for a subproject with no meson.build)
- [ ] Write `meson.build` at the repo root:
  - [ ] `project('nethack', 'c', version: '3.7', default_options: ['c_std=c11'])`
  - [ ] Platform detection for Linux / macOS / Windows conditionals
  - [ ] Lua sources compiled as a static library
  - [ ] System curses dependency on Linux/macOS (`dependency('ncurses')`)
  - [ ] pdcursesmod subproject on Windows
  - [ ] NetHack core sources → static lib or object list
  - [ ] TTY windowport sources
  - [ ] Final `nethack` executable target
- [ ] Handle `makedefs` — NetHack generates `pm.h`, `onames.h`, and
      `date.h` at build time via `util/makedefs`. Wire this as a Meson
      custom target that runs before the main compile.
- [ ] Verify `meson setup builddir && ninja -C builddir` produces a
      working `nethack` binary on Linux
- [ ] Smoke test: run `./builddir/nethack -u Foo` and reach the TTY game
      screen without a crash

### Source references

- `sys/unix/hints/linux.370` — authoritative Linux source/flag list
- `sys/unix/hints/macOS.370` — authoritative macOS source/flag list
- `sys/unix/Makefile.top`, `Makefile.src`, `Makefile.utl` — existing
  build logic to translate into Meson
- `submodules/lua/` — Lua source (after submodule init)
- `include/` — all headers; add as include directory

### Complexity notes

- `makedefs` must run before any `.c` file that includes `onames.h` or
  `pm.h`. Meson's `custom_target` with `depfile` support handles this;
  do not skip it or you will get intermittent missing-header errors.
- NetHack's source list is not a simple glob — some files are
  conditionally included based on windowport. Start from the hints file,
  not from `ls src/`.
- Do not add Windows or macOS support until Linux builds cleanly. One
  platform at a time.
- Do not add tiles or sound to this task. That is N2 and N3.
- The existing `azure-pipelines.yml` is not a useful reference for Meson
  — it uses the old Makefile system. Ignore it.

### Done when

- `meson setup builddir && ninja -C builddir` completes without error on
  Linux
- The resulting `nethack` binary launches and reaches the TTY game screen
- No source files from the old Makefile system are referenced in
  meson.build (it is a clean rewrite, not a wrapper around make)
- meson.build is readable and commented enough that adding a new
  windowport (hack2bridge) is a localised addition, not a sprawling change
