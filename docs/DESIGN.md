# Protohack — Design

> A protocol for turn-based, grid-oriented games. A specification, not an
> engine. A contract between front ends and back ends.

This document captures the foundational design philosophy of Protohack —
the *why* behind the protocol, not the protocol itself. For the wire format
and event vocabulary, see `PROTOCOL.md`.

---

## The name

The exact origin of the name has been lost, but several readings all fit
and were probably all in play at once:

- **protocol + hack** — a protocol for hack-style (roguelike) games, the
  most literal reading
- **proto + hack** — something foundational that underlies hack-style
  games; *proto* in the sense of "first" or "base layer"
- **prototype + hack** — a prototype protocol, a sketch of what a
  hack-style game infrastructure could be before it becomes a real product

None of these are wrong. The name was coined during early design and the
specific intent was not written down in time. If you remember, update this
section.

---

## What Protohack is

Protohack is a **protocol specification**. It defines a contract between two
sides:

- **Engines** — programs that simulate a game world and emit events
  describing it.
- **Frontends** — programs that render those events for a human player and
  send back the player's input.

Protohack is *not* an engine. It is *not* a frontend. It is *not* a game.
It is the wire format and event vocabulary that lets engines and frontends
talk to each other.

Implementations of either side that conform to the spec are *Protohack-
conformant*. Anyone — present authors, future authors, third parties — can
write a conformant engine or frontend without coordinating with anyone else.
That's the lever the protocol provides.

The closest analogues are LSP, MCP, or X11 — protocols whose value lives in
the contract, not in any single implementation, and which outlive the
implementations that birthed them.

---

## What Protohack is for

The target domain is **turn-based grid games**: roguelikes, tactics games,
dungeon crawlers, hex wargames, and adjacent forms. Games where:

- The world is a grid (square, hex, or otherwise discrete).
- Time advances in turns, not real-time.
- The frontend's job is to render state and collect input, not to simulate.
- The engine's job is to simulate, not to render.

This domain is broad enough to cover NetHack, FOXHOLE (a planned WWII
tactics game), traditional hex wargames, and most things in the Slitherine
catalogue. It is narrow enough that the protocol can be opinionated about
what a "cell" is, what a "turn" is, what a "menu" is.

The protocol does not target real-time games, 3D games, or games where the
simulation and presentation are deeply entangled. Those domains have their
own solutions and are well-served by existing engines.

---

## Why a protocol, not an engine

The default architecture for a game is monolithic — engine and frontend
compiled together, sharing memory, sharing types. This is fine for one
game. It scales badly for an ecosystem.

A protocol-based architecture has properties that matter for a long-lived
project:

**Implementations are independently replaceable.** A new frontend can be
written without touching the engine. A new engine can be written without
touching any frontend. This is the difference between "we're stuck on
GTK 2 forever" and "we have five frontends and you can use whichever one
you like."

**Implementations can be in different languages.** The engine can be C, the
frontend Python; the engine Rust, the frontend Swift. The protocol is the
lingua franca.

**The contract is inspectable.** Wire-protocol traffic can be logged,
replayed, diffed, and used for debugging. Compare to introspecting a
shared-memory game state, which is harder.

**Modding is structural, not bolted on.** A modder writing a new frontend
isn't fighting an engine API designed for one specific frontend. They're
implementing a published spec, the same spec the reference frontend
implements.

**AI agents fit naturally.** A Protohack engine emits structured events; a
Protohack frontend consumes them. An AI dungeon master, an AI player, an AI
test harness — all of these are just programs that speak the protocol.
There's nothing to bolt on.

The cost of this architecture is real: a wire protocol is more work to
design and evolve than an in-process API, and it imposes serialization
overhead. The cost is paid once, by the spec authors. The benefits accrue
forever, to everyone who builds on the spec.

---

## The ecosystem model

Protohack aims to enable an ecosystem, in roughly the sense that the web is
an ecosystem rather than a product. The value of HTTP is not that it is
cleverly designed; the value is that everyone agrees to speak it. Anyone
can write a server, anyone can write a browser, and they all interoperate.

The reference points for the *kind* of games Protohack hopes to host are
publishers like Slitherine: *Field of Glory*, *Order of Battle*, *Panzer
Corps*, *Strategic Command*, *Armoured Brigade*. These games share DNA —
hex or square grids, turn-based, deeply systemic, data-driven, lean on
flashy graphics, deep on simulation. They feel like a family even when the
underlying tech differs. Protohack wants to enable that family feeling, but
through a shared protocol rather than a shared publisher or a shared
engine.

This implies a few things about scope:

- Protohack is **frontend-agnostic on visual style.** The protocol carries
  semantic information ("this unit is suppressed", "this tile has heavy
  cover"). The frontend chooses how to render it. There is no canonical
  Protohack visual style.

- Protohack is **engine-agnostic on simulation.** The protocol does not
  specify how the engine schedules turns, generates dungeons, or resolves
  combat. It specifies what the engine tells the frontend about those
  things.

- Protohack is **game-agnostic on rules.** The protocol carries a
  vocabulary general enough to describe NetHack, FOXHOLE, and hypothetical
  future games without privileging any one of them.

The risk in platform-building is well-known: projects spend years on the
platform and never ship a game. The mitigation is to **let one game pull
the platform forward.** Hack2 — a NetHack frontend speaking Protohack — is
that lead title. Every protocol decision is tested against "does this work
for NetHack?" before being added. Speculative protocol features can be
designed but should not be implemented until a real game needs them.

---

## NetHack as the v1.0 baseline

The conformance target for Protohack v1.0 is: **a Protohack-conformant
engine and frontend pair must be able to play NetHack to completion.**

This is a strong baseline, not a weak one. NetHack exercises a remarkable
fraction of the UI surface area a turn-based grid game can want:

- Map updates with cell-level granularity
- Glyph and tile dual representation
- Message stream with importance levels and `--More--` pacing
- Three input modes: single key, yes/no, free-text line input
- Rich menu protocol: single-select, multi-select, paginated, with letter
  shortcuts and category headers
- Status line with discrete fields (HP, AC, level, gold, hunger,
  encumbrance, status effects, dungeon level)
- Game lifecycle: handshake, version negotiation, save/load/quit, game over

There is essentially no UI primitive a tactical or roguelike game needs
that does not have a NetHack analogue. Even line-of-fire targeting maps
onto NetHack's directional prompt and travel cursor. Picking NetHack as the
baseline does not constrain Protohack to NetHack-shaped games; it ensures
that Protohack can drive *any* game whose UI is at least as rich as
NetHack's.

The corollary is a **concrete conformance test that lasts forever.** A
Protohack-conformant frontend is one that can play NetHack to completion. A
Protohack-conformant engine is one whose output a NetHack-trained player
finds legible and complete. If a future spec change breaks NetHack
playability, the spec change is wrong.

What NetHack does *not* exercise — and what therefore lives in v1.x or
v2.0, not v1.0 — includes:

- Multi-unit selection and command (NetHack has one player; tactics games
  have squads)
- Rich tile metadata (cover state, elevation, suppression, fog-of-war
  level)
- Animation and timing hints (NetHack is essentially instantaneous between
  turns)
- Camera and viewport hints (NetHack assumes the frontend handles its own
  viewport)

These are real and will need to be added. But they will be added *after*
v1.0 ships, after a real tactics game is being built, and after the actual
shape of the need is understood. Speculative extension is fine;
speculative *core* is dangerous.

---

## Tile-first, ASCII-permitted

Traditional roguelike orthodoxy treats ASCII as primary and tiles as a
presentation skin. NetHack itself is built that way: the engine thinks in
glyphs, and tile rendering is essentially a glyph-to-tile lookup table done
by the windowport. That is why NetHack tile support has always felt
slightly grafted-on; the engine does not really know about tiles.

Protohack inverts this. The protocol is **tile-first with ASCII as a
permitted fallback.** Specifically:

> Protohack is a tile-oriented protocol. Engines MUST send a tile index for
> every cell. Engines MUST also send a glyph for every cell, to support
> ASCII frontends and to provide a stable semantic identifier across
> tilesets. ASCII frontends are valid Protohack implementations and MUST be
> supported by the spec, but the protocol's vocabulary is not constrained
> to what ASCII can express. Where tile and glyph representations diverge
> in expressive power, the tile representation is canonical; the glyph is
> a best-effort fallback.

This decision has real consequences:

**Tile-capable frontends are first-class.** They consume tile indices
directly. No glyph→tile lookup table maintenance, no "what does `\` mean in
this context" ambiguity.

**ASCII frontends still work** by ignoring the `tile` field and rendering
`glyph`. They are valid implementations, just not privileged ones.

**The engine speaks both simultaneously.** This is cheap on the wire — one
extra integer per cell — and it removes a class of "the ASCII says one
thing and the tile says another" bugs that have plagued NetHack tile ports.

**Tile-only events are permissible.** Animated muzzle flashes, expanding
smoke clouds, unit facing directions, suppression overlays — these have no
good ASCII representation. In a tile-first world, the protocol can carry
this richer state directly. ASCII frontends render what they can and skip
what they cannot.

**Subtype disambiguation is cleaner.** A `subtype` field exists for cases
where ASCII vocabulary is too thin (e.g., `#` as wall vs. corridor). In a
tile-first world, the tile index *is* the subtype for tile frontends; the
subtype field remains load-bearing only for ASCII rendering.

The glyph is not deprecated. It remains the **stable semantic token**:
`@` means "the player," not "render an at sign." Tools that are not
rendering — accessibility readers, AI agents reading game state, save-game
inspectors, debuggers, log analyzers — depend on glyphs as a portable
vocabulary. The glyph being a stable cross-engine identifier is more
important than the glyph being a renderable character.

Strict ASCII-only rendering is a valid Protohack frontend. Tile-only
rendering is also valid. The protocol does not pick a winner; it picks a
canonical representation when they diverge.

---

## Naming and hierarchy

The naming hierarchy reflects the architectural layering:

- **Protohack** — the protocol specification. The thing this document is
  about. Versioned independently. Has no executable code of its own.

- **Hack2** — a NetHack frontend (and the bridge windowport that makes
  NetHack speak Protohack). The reference implementation of the engine
  side for v1.0.

- **FOXHOLE** — a planned WWII tactics game built on a future
  Protohack-conformant engine. Not yet implemented.

- **Future engines and frontends** — named separately, on their own merits,
  not derivative of "Protohack" or "Hack2".

The protocol name does not appear in the names of games or frontends, the
same way "HTTP" does not appear in "Wikipedia." This is intentional and
correct. A game built on Protohack is a game, not a Protohack-thing.

---

## Spec discipline

A protocol is only as good as the discipline of its evolution. A few
commitments that follow from the architecture:

**The spec is the canonical artifact.** `PROTOCOL.md` is the project.
Engines, frontends, and games are reference implementations or example
material. Spec changes are the most consequential changes in the
ecosystem.

**At least two implementations on each side.** A protocol with one engine
and one frontend is an internal API with delusions of grandeur. The
discipline that makes a protocol real is having a second implementation
that surfaces the assumptions the first was making. Hack2 already has two
frontend implementations (Textual TUI and pygame); a second engine, even
a toy one, will be needed to validate v1.0.

**Versioning from day one.** A `protocol.version` field in the handshake.
Semantic versioning for the spec itself. A clear policy on breaking vs.
additive changes. LSP got this wrong early and is still paying for it; MCP
learned from LSP. Protohack should learn from both.

**The spec runs slightly ahead of implementations.** Events and commands
that no current implementation uses are permitted in the spec if their
shape is clear and their need is foreseeable. Protocols benefit from being
designed for the medium term, not retrofitted from whatever the lead
implementation happened to do.

**Conformance is a binary claim.** An implementation either conforms to a
spec version or it does not. Partial conformance is not a category. The
NetHack-playability test is the practical check.

---

## What this document is not

This document is the architectural philosophy. It is deliberately separate
from:

- `PROTOCOL.md` — the wire format and event vocabulary (the spec itself).
- `ARCHITECTURE.md` — how the reference engine and frontends are
  organized internally. Distinct from what the spec requires.
- `AGENTS.md` — how AI coding agents should work within the project.
- `AI_DM.md` — how the protocol supports AI dungeon mastering, a forcing
  function for getting the event vocabulary right.

When in doubt, this document answers *why*. The other documents answer
*what* and *how*.
