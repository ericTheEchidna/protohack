# DESIGN.md patch — add this section

This is the section that replaces the dropped `AI_DM.md`. It belongs in
`DESIGN.md`, inserted between **"Tile-first, ASCII-permitted"** and
**"Naming and hierarchy"**.

Just paste the section below into DESIGN.md at that point. No other
changes to DESIGN.md are needed.

---

## Semantic richness

A protocol can carry presentation ("render this glyph at this cell") or
semantics ("a hostile orc with low HP is adjacent to the player"). These
are different commitments with different consequences.

Protohack commits to **semantic richness** as a design principle. Where a
choice exists between sending presentation data and sending semantic data,
the protocol prefers semantic data, with presentation derived by the
frontend.

Concretely:

- A tile event carries the entity *type* (orc, player, door), not just
  the rendered appearance. A frontend can choose to render an orc
  differently based on hostility, HP, or status — but only because the
  protocol gave it that information, not because it inferred it from a
  tile index.
- A status event carries discrete fields (`hp.current`, `hp.max`,
  `hunger`, `encumbrance`) rather than a pre-formatted status line
  string. Frontends choose layout.
- A message event carries an importance level or category, not just text.
  An accessibility frontend can announce critical messages differently
  from flavor text; an AI consumer can filter by category.

The forcing function for this principle is asking: **could a non-rendering
consumer of the protocol use the game state intelligently?** A debugger,
a save-game inspector, an AI player, a screen reader, a replay analyzer
— all of these consume the wire format without rendering it. If the
protocol only carries presentation data, none of them can do their job
without reverse-engineering meaning from glyphs and pixels. If the
protocol carries semantic data, all of them work.

This is also what makes the protocol future-proof. A purely presentational
protocol locks every consumer into the original frontend's assumptions
about what matters. A semantic protocol lets future frontends and tools
make different choices about what to surface, what to hide, and how to
present what they show — without the engine ever having to change.

The cost is that the protocol's vocabulary is larger and its design
requires more thought. Each new event type has to be designed with both
"how does the reference frontend render this?" *and* "what would a
non-rendering consumer want to know about this?" in mind. The cost is
worth paying. A presentation-only protocol is a temporary convenience; a
semantic protocol is infrastructure.
