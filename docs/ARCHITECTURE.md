# Architecture

> How the reference implementations are organized. This document describes
> *what we built*, not *what the protocol requires*. For the protocol spec,
> see `PROTOCOL.md`. For the philosophy behind the protocol, see
> `DESIGN.md`.

Protohack is a specification, not an engine. But specifications need
reference implementations, and the reference implementations have
architectural choices worth documenting — both for contributors working on
them and for authors of new implementations who want a known-good shape to
crib from.

This document covers three reference implementations:

- **Hack2** — a NetHack windowport that speaks Protohack (the engine side)
- **Hack2 Textual frontend** — a Python TUI frontend
- **Hack2 pygame frontend** — a Python tile-based frontend

None of what follows is normative. A conformant implementation can be
structured however it likes. These are the shapes the reference code
happens to take, and the reasons.

---

## Repository layout

The current development tree lives at `Hack2-1/` and contains both the
Protohack docs/spec and the NetHack windowport in a single repo. This is
a transitional state. The plan is to split into two repos: a Protohack
repo (spec, docs, frontends) and a NetHack fork repo (the simulation
engine plus the hack2bridge windowport). Once split, frontends will
locate the engine binary via `HACK2_ENGINE` exactly as they do now —
nothing in the protocol or frontend code depends on the colocation.

Current layout:

```
Hack2-1/
├── DESIGN.md          # philosophy
├── PROTOCOL.md        # the spec
├── ARCHITECTURE.md    # this document
├── AGENTS.md          # AI coding agent instructions
├── nethack/           # vendored NetHack source + hack2bridge windowport
│   └── win/
│       └── hack2bridge/    # Protohack-speaking windowport
└── frontends/
    ├── textual/       # Python Textual TUI frontend
    └── pygame/        # Python pygame frontend
```

Planned split:

```
protohack/                     # spec + frontends
├── DESIGN.md
├── PROTOCOL.md
├── ARCHITECTURE.md
├── AGENTS.md
└── frontends/
    ├── textual/
    └── pygame/

nethack-protohack/              # NetHack fork
└── win/
    └── hack2bridge/
```

The `nethack/` tree is a fork of upstream NetHack with the `hack2bridge`
windowport added alongside the existing `tty`, `curses`, `X11`, etc.
ports. The fork is kept merge-friendly with upstream so DevTeam fixes
can be pulled in.

---

## Hack2 — the NetHack windowport engine

### Role

Hack2 is the engine half of the reference Protohack pair. Concretely, it is
NetHack compiled with the `hack2bridge` windowport selected. The binary
runs as a subprocess of the frontend, reads commands on stdin, and emits
events on stdout.

```
    ┌─────────────┐  stdin  (commands)  ┌──────────────────┐
    │  Frontend   │────────────────────>│  nethack         │
    │  (any)      │<────────────────────│  -whack2bridge   │
    └─────────────┘  stdout (events)    └──────────────────┘
```

### Why a windowport, not a fork

NetHack already abstracts its UI behind `struct window_procs` — a function
table the engine calls into for everything that touches the player. The
existing tty, curses, and X11 windowports are all implementations of that
table. Adding `hack2bridge` as a fourth windowport is the path of least
resistance and the path of greatest forward-compatibility:

- The simulation logic is untouched. NetHack's combat, dungeon generation,
  monster AI, and item interactions remain exactly as the DevTeam wrote
  them. Decades of "yet another stupid death" balance is preserved.
- Upstream patches merge cleanly. We are not maintaining a parallel
  simulation.
- The conformance test ("can play NetHack to completion") is automatic by
  construction. If the windowport faithfully implements `window_procs`,
  NetHack works; if it doesn't, NetHack breaks visibly.

### Structure

The hack2bridge windowport lives in `nethack/win/hack2bridge/` alongside
the other ports. Files map to the standard NetHack windowport layout:

- `winhack2.c` — implementation of `struct window_procs` (the entry points
  NetHack core calls into)
- `proto_emit.c` — wire-protocol serialization helpers (emits `< ns.event
  k=v` lines on stdout)
- `proto_parse.c` — wire-protocol parsing for inbound `> command` lines on
  stdin
- `winhack2.h` — internal interface

The boundary between `winhack2.c` and `proto_emit.c` is the same shape as
the boundary between `Game` and `ProtocolWriter` in the C++ engine work:
windowport logic decides *what* to emit, the protocol module decides *how*
to serialize it. This keeps the wire format changes contained to one file.

### State held by the windowport

The windowport holds minimal state of its own. NetHack core owns the game
state; the windowport is a translator. The state it does hold is:

- **Cell dirty tracking** — which (x, y) cells have changed since the last
  flush, to avoid retransmitting the whole map every turn.
- **Menu construction state** — the in-progress menu being built between
  `start_menu` and `end_menu` calls (NetHack's menu API is incremental).
- **Pending input** — input characters NetHack core has requested but not
  yet consumed.

Everything else (player stats, inventory, dungeon level, monster
positions, etc.) lives in NetHack core's own data structures and is
serialized to the wire on demand.

### Build modernization

The reference build has been modernized off NetHack's traditional
`sys/unix/` Makefile soup. The hack2bridge port builds via CMake with the
following targets:

- Linux x86_64 / arm64
- macOS x86_64 / arm64
- Windows x86_64 (MSVC and MinGW)

GitHub Actions produces release binaries on every tag. The goal is that a
new contributor can `git clone && cmake -B build && cmake --build build`
and get a working `nethack -whack2bridge` binary, on any platform, without
fighting platform-specific config.

This is intentionally ahead of upstream's build situation. Patches to
unify the build systems may eventually be offered upstream; for now,
keeping the modernization local to the fork is simpler.

---

## Frontends

The two reference frontends are written in Python. This is a deliberate
choice for the reference implementations:

- Python is approachable for contributors.
- The frontends are I/O-bound, not CPU-bound. Performance is not a
  concern.
- Two frontends in the same language make it easier to share patterns
  (parsing, subprocess management) without sharing code, which is the
  right balance for testing protocol assumptions.

A future native or web frontend in another language is encouraged and
expected. The frontends below are not the canonical frontends; they are
the frontends that happen to exist.

### Shared concerns

Every frontend has the same general shape:

```
    ┌──────────────────────────────────────────────────┐
    │ Frontend                                         │
    │                                                  │
    │  ┌────────────────┐    ┌──────────────────────┐  │
    │  │ Engine subproc │───>│ Event queue          │  │
    │  │ manager        │    └──────┬───────────────┘  │
    │  └────────────────┘           │                  │
    │         ▲                     ▼                  │
    │         │              ┌──────────────────┐      │
    │         │              │ State model      │      │
    │         │              │ (cells, msgs,    │      │
    │         │              │  menus, status)  │      │
    │         │              └──────┬───────────┘      │
    │         │                     │                  │
    │         │                     ▼                  │
    │         │              ┌──────────────────┐      │
    │         │              │ Renderer         │      │
    │         │              └──────────────────┘      │
    │         │                                        │
    │  ┌──────┴─────────┐                              │
    │  │ Input handler  │                              │
    │  └────────────────┘                              │
    └──────────────────────────────────────────────────┘
```

Concrete responsibilities:

- **Engine subprocess manager** — spawns the engine binary, manages its
  lifecycle (start, restart, stop), reads its stdout into a thread-safe
  queue, writes commands to its stdin. Owns the `HACK2_ENGINE` env var.
- **Event queue** — decouples the I/O thread from the render thread. The
  reader thread blocks on stdout; the main loop drains the queue each
  frame.
- **State model** — accumulates cell updates, message history, menu state,
  status fields. Survives across frames; mutated by inbound events.
- **Renderer** — reads from the state model, draws the screen. Pure
  presentation.
- **Input handler** — translates user input (key events, mouse) into
  Protohack `> command` lines and writes them to the engine.

The send and readline points (`EngineProcess.send()` and `readline()` in
the Textual frontend; `Engine.send()` and the reader thread in pygame) are
the I/O intercept points for any debugging, replay, or protocol logging.

### Textual TUI frontend

Lives at `frontends/textual/`. Built on the [Textual](https://textual.textualize.io/)
framework.

Notable points:

- Uses the `HACK2_ENGINE` environment variable to locate the binary.
- A `_pre_log` pre-mount buffer is required for handshake lines emitted
  before the Textual app finishes mounting widgets — without it, early
  events are lost.
- Map updates are batched until a `map.end` event before triggering a
  refresh. Per-cell refreshes during initial map generation cause visible
  tearing and unacceptable redraw cost.
- Player rendering uses reverse video rather than `bold=True`, which on
  Textual produces a synthesized-bold artifact for the `@` glyph.

### pygame frontend

Lives at `frontends/pygame/`. Files:

- `main.py` — entry point, subprocess management, main loop, top-level
  rendering of map / message bar / debug panel
- `protocol.py` — wire-protocol parser and command formatters
- `tileset.py` — tile atlas helper, blits 16×16 tiles by index from a BMP
- `input.py` — pygame KEYDOWN to NetHack key character mapping
- `prefs.py` — modal preferences dialog (font selection, font size)
- `config.py` — JSON-backed persistent preferences, stored at
  `$XDG_CONFIG_HOME/hack2/prefs.json`

Layout (current development state):

```
   ┌────────────────────────────────────────┐
   │ Menu bar (24px)                        │
   ├────────────────────────────────────────┤
   │                                        │
   │ Map (1280 × 336, 80 cols × 21 rows)    │
   │ rendered from tileset                  │
   │                                        │
   ├────────────────────────────────────────┤
   │ Message bar (64px, 3 lines)            │
   ├────────────────────────────────────────┤
   │ Debug protocol log (145px)             │
   └────────────────────────────────────────┘
```

The target layout is 1920×1080 fullscreen with a 1280px map viewport on
the left and a 640px character sheet panel on the right; the current
development layout is a transitional shape used while menu, status, and
character-sheet handling are being implemented.

Notable points:

- The engine subprocess is owned by an `Engine` class in `main.py`. A
  daemon thread reads stdout into an unbounded `queue.Queue`; the main
  loop drains the queue each frame.
- Only changed cells are redrawn, but the full screen is flipped each
  frame. The map itself is buffered as a `dict[(x, y), (glyph, tile_idx)]`
  in `cell_buf`.
- `map.clear` events do *not* clear the cell buffer. NetHack redraws only
  changed cells after a clear, and a naive clear would leave the frontend
  with a black map until the next full refresh.
- The font preference reload path rebuilds the menu bar, because the menu
  bar caches font metrics. This is a known coupling and could be cleaner.

### Frontend non-goals

These frontends deliberately do *not*:

- Hold authoritative game state. The engine is the source of truth.
- Implement any game logic. Even trivial things like "is this cell visible
  to the player" come from the engine.
- Smooth over protocol gaps with frontend heuristics. If the engine does
  not send the information, the frontend does not display it.

The temptation to add "small" frontend-side intelligence is constant and
should be resisted. Every piece of game knowledge in the frontend is a
piece of knowledge that another frontend will have to reimplement, and a
piece of knowledge that gets out of sync with the engine.

---

## Future architectural work

Captured here so it doesn't get lost. None of these are commitments;
they're the shape of the next several rounds of work as currently
understood.

**Engine side:**

- A second, non-NetHack reference engine. Even a 200-line toy engine that
  walks an `@` around a room would sharpen the spec by surfacing
  NetHack-shaped assumptions baked into the current windowport.
- The C++ engine work (Hack2 native, with the World → Region → GameMap →
  Level → Grid → Cell hierarchy) is in progress as a separate effort and
  will eventually be a second Protohack-conformant engine.

**Frontend side:**

- The pygame frontend is blocked on the menu protocol — inventory,
  spellbook, and any other multi-item selection are unreachable until the
  spec and implementation land.
- A web frontend is plausible long-term, both for distribution and as a
  third implementation to validate the spec.
- Animation and timing-hint support, when the protocol gains it.

**Cross-cutting:**

- A protocol replay tool: read a logged `< ns.event` stream, drive a
  frontend with it. Useful for testing frontends without running the
  engine, and for sharing bug reproductions.
- A protocol fuzzer for the engine side: feed malformed `> command` lines
  and verify the engine fails gracefully.
- A conformance test harness: a scripted NetHack playthrough that any
  frontend can be checked against.
