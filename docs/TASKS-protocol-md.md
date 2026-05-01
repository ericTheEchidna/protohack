Protohack
Generate PROTOCOL.md spec from Hack2 windowport source

## Phase D1 — Wire Protocol Specification

Extract the canonical wire protocol from the Hack2 NetHack windowport
source and produce `PROTOCOL.md` as the v1.0 specification document. This
is reverse-engineering from the working reference implementation, not
clean-room design. Every event and command in the spec must be grounded
in real emit/parse code in the windowport. Gaps where the implementation
is incomplete (notably menus, status line) should be flagged as TBD
sections rather than invented.

### Tasks
- [ ] Read `DESIGN.md` and `ARCHITECTURE.md` first to understand the
      protocol's role and the tile-first / NetHack-baseline commitments
- [ ] Enumerate every `< ns.event` emission in the windowport
  - [ ] Grep `/home/eric/source/Hack2-1/nethack/win/hack2bridge/` for
        emit calls (likely `proto_emit_*` or similar serialization
        helpers)
  - [ ] For each event: record the namespace, event name, every
        parameter key, parameter type (string/int/bool), and the
        triggering condition in NetHack core
- [ ] Enumerate every `> command` parser path
  - [ ] Grep the windowport for inbound parsing (likely `proto_parse.c`
        or equivalent)
  - [ ] For each command: record the verb, every parameter key, type,
        and what NetHack core action it triggers
- [ ] Cross-check against the frontend reference implementations
  - [ ] `frontends/textual/protocol.py` — what does it parse and emit?
  - [ ] `frontends/pygame/protocol.py` — same check
  - [ ] Any event the frontends parse but the engine doesn't emit, or
        vice versa, gets flagged as a divergence
- [ ] Draft `PROTOCOL.md` with these sections:
  - [ ] Introduction — one paragraph, points to DESIGN.md for philosophy
  - [ ] Wire format — `< ns.event k=v` framing rules, quoting,
        escaping, line termination
  - [ ] Direction conventions — `<` engine-to-frontend, `>`
        frontend-to-engine
  - [ ] Versioning — protocol.version field, semver policy, what counts
        as breaking vs additive
  - [ ] Handshake — the startup exchange (engine identifies itself,
        version negotiation)
  - [ ] Namespace catalog — one section per namespace (`map`, `msg`,
        `player`, `status`, `game`, etc. as found in source)
  - [ ] For each event/command in each namespace: name, direction,
        params table, semantics, examples
  - [ ] Conformance — the "can play NetHack to completion" test,
        MUST/SHOULD/MAY language for each event
  - [ ] Gaps and TBD — explicit list of unfinished areas (menus, status
        line, anything else surfaced during enumeration)
- [ ] Use RFC 2119 keywords (MUST, SHOULD, MAY, MUST NOT) consistently
- [ ] Mark tile-first canonical, ASCII permitted, per DESIGN.md
- [ ] Verify every documented event has a real source-line reference
      (file + approximate line) in a comment or footnote

### Source
- `/home/eric/source/Hack2-1/nethack/win/hack2bridge/` — the windowport.
  Primary source of truth for what the engine actually emits and
  accepts. Read every `.c` and `.h` file in this directory. Path is
  current absolute location; will move once NetHack is forked into a
  standalone repo.
- `/home/eric/source/Hack2-1/nethack/include/winprocs.h` — NetHack's
  `struct window_procs` defining what the windowport must implement;
  helps map windowport functions to protocol events
- `/home/eric/source/Hack2-1/frontends/textual/protocol.py` — Textual
  frontend's parser. Validates event vocabulary from the consumer side.
- `/home/eric/source/Hack2-1/frontends/pygame/protocol.py` (~30 lines) —
  pygame frontend's parser. Same.
- `/home/eric/source/Hack2-1/frontends/pygame/main.py` — search for
  `parse_engine_line` callers to see which events the pygame frontend
  actually handles vs. ignores
- `DESIGN.md` — the philosophy. Read first.
- `ARCHITECTURE.md` — reference implementation structure. Useful
  context.

### Complexity Notes
- The windowport is incomplete; do not invent events to fill gaps. Menus
  and status line are known unfinished. Document them as TBD with a
  sketch of the expected shape, not as completed spec.
- NetHack's `window_procs` has functions that the hack2bridge port may
  stub out or partially implement. A stub is not a spec event. Only
  document what the port actually emits over the wire.
- Distinguish event vocabulary (what the engine sends) from frontend
  state (what the frontend accumulates). The protocol does not specify
  frontend state, only the events that mutate it.
- The KV format uses `key=value` with quoted strings for values
  containing spaces. Document the exact escape rules from the parser
  regex in `protocol.py` — do not paraphrase.
- `map.clear` semantics are subtle: NetHack redraws only changed cells
  after a clear, so frontends must NOT reset their cell buffer on
  `map.clear`. Document this explicitly; it has bitten frontend
  implementations.
- Render batching (`map.end` triggering frontend refresh) is a frontend
  concern, not a spec requirement, but the spec should note that
  frontends MAY batch and that engines SHOULD emit `map.end` after burst
  updates.
- The pygame frontend currently treats `msg.yn` and `msg.getlin` as
  appending to the message log. Confirm this matches what the engine
  emits before documenting.
- Player input commands (`player.key`, `player.line`) need careful
  treatment of escape characters: `\r`, `\033`, `\010` are passed through
  as literal bytes. The spec must document the bytes-on-the-wire
  encoding.
- Avoid AI-shaped over-engineering of the spec doc itself — no
  speculative sections, no "future considerations" beyond the explicit
  TBD list.

### Done When
- `PROTOCOL.md` exists at the repo root and documents every event and
  command currently implemented in the hack2bridge windowport
- Every event/command in the spec is traceable to a specific source file
  and approximate line in the windowport or frontends
- Tile-first / ASCII-permitted is stated explicitly with MUST/SHOULD
  language
- Versioning, handshake, and wire format sections are concrete enough
  that a third-party implementer could write a conformant frontend from
  the document alone
- Known gaps (menus, status line) are listed in a "Gaps and TBD" section
  rather than silently omitted
- The `< ns.event k=v` framing rules including quoting and escaping are
  documented from the parser regex, not paraphrased
- A reader who has not seen the source can identify which events are
  engine-to-frontend vs frontend-to-engine without ambiguity
