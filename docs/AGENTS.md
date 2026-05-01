# AGENTS.md

> Instructions for AI coding agents (Claude Code, Cursor, etc.) working in
> this repository. Humans should also read this — most of it is good
> advice for human contributors too — but the document is written
> specifically to address the failure modes AI agents tend to fall into on
> this codebase.

## Read first, write second

Before changing anything, read:

1. `DESIGN.md` — the architectural philosophy. Understand that Protohack
   is a *protocol*, not an engine.
2. `PROTOCOL.md` — the wire spec. The canonical artifact of the project.
3. `ARCHITECTURE.md` — how the reference implementations are organized.
4. The specific source files involved in the task.

Do not skip these. The codebase has a specific shape and a specific
philosophy, and reading the docs is the fastest way to align with both.
"Reasonable defaults" derived from training data on generic codebases
will produce code that gets reverted.

## The fundamental commitment

Protohack is a **specification** with **reference implementations**. The
spec is the product. The implementations exist to validate the spec and
to ship working software.

This has two consequences for any code change:

1. **Spec changes and implementation changes are different things.**
   Changing the wire protocol is a spec change and requires updating
   `PROTOCOL.md`. Changing how the pygame frontend renders the message
   bar is an implementation change and does not. Do not silently mix
   them.

2. **Implementation choices that contradict the spec are bugs.** If the
   spec says cells carry both `glyph` and `tile`, an engine that only
   emits `tile` is broken. If the spec says ASCII frontends are valid,
   a frontend that requires tiles to render is non-conformant.

When in doubt, the spec wins. If the spec is wrong, fix the spec — but
fix it explicitly, not by quietly diverging in the code.

## Code style — "more like C than C++"

The C++ engine work has a specific style discipline. AI-generated C++
tends to introduce excessive complexity; this codebase actively pushes
back against that.

Avoid, unless there is a specific concrete reason to use them:

- `std::optional` — prefer sentinel values, out-parameters, or restructured
  control flow
- `std::move` — the cases where it matters are rare; do not pepper it
  through every function signature
- Anonymous namespaces — use `static` at file scope
- Nested `std::vector<std::vector<T>>` — flatten with a stride, or use a
  proper grid type
- Factory methods and factory classes — direct construction is usually
  fine
- SAT collision detection and similar over-engineered geometry — grids
  are grids, AABBs are AABBs
- Inheritance hierarchies more than two levels deep — composition or flat
  type unions are usually clearer
- `auto` for non-obvious types — name the type when it aids reading

Prefer:

- Plain structs and free functions
- Explicit ownership (raw pointers and references where lifetime is
  clear, `std::unique_ptr` where it isn't, almost never `std::shared_ptr`)
- Clarity over cleverness
- Small files and small functions

When in doubt, look at how `Game.cpp`, `ProtocolWriter`, and `Parsing.h`
are written, and match their shape.

## Beware premature abstraction

The codebase has a history of rolled-back abstractions that seemed
reasonable in isolation but added complexity without paying for itself.

A canonical example: a `terrain_type` enum was added to `Cell` to
distinguish corridors from floor tiles. This was rolled back. The
distinction belongs in `Level::cell_info_at()` by checking room bounds
against `level.rooms` — it's a derived property, not an intrinsic one,
and forcing it onto the `Cell` type meant every code path that touched
cells had to think about terrain types.

Before introducing a new field, type, or class, ask:

- Is this distinction intrinsic to the entity, or derivable from existing
  data?
- Will every code path need to handle every value of this new
  distinction, or only a few?
- Is there an existing pattern (e.g., `Door`/`doors` for stairs) that
  already covers this case?

When the answer points to "derive it" rather than "add a field," derive
it.

## NetHack glyphs are domain tokens

Single-character glyphs in this project are not arbitrary presentation
choices. They are a semantic vocabulary — `@` means "the player," not
"render an at sign." This vocabulary is portable across engines and
frontends, and is read by tools that aren't rendering (accessibility,
AI agents, log analyzers).

When extending the glyph vocabulary:

- Reuse existing NetHack glyphs where the meaning is established.
- Add a `subtype` field for cases where the glyph is genuinely ambiguous
  (e.g., `#` as wall vs. corridor). Do not invent new glyphs to
  disambiguate.
- For presentation-only distinctions that ASCII can't carry (animations,
  overlays, fine-grained state), use protocol fields beyond `glyph` and
  `tile`. Glyphs stay stable; new state lives in new fields.

## The TASKS.md format

Hand-offs to AI coding sessions use a specific structured format.
Documents are written for direct copy-paste into a Claude Code session
and are sized to fit one session. Sections include:

- Context — what the task is part of, what the session needs to know
- Source file references — exact paths, with line numbers where relevant
- Task checklist — concrete steps, ordered
- Complexity notes — known traps and prerequisites
- Done-when criteria — explicit, checkable

If you are an AI agent generating a TASKS.md, follow this format. If you
are an AI agent receiving a TASKS.md, treat its done-when criteria as
the contract for the session — do not declare the task complete unless
they are met.

## Feedback discipline

The maintainer communicates tersely and technically. Expect:

- Design direction without elaboration; you are expected to flesh out the
  details and propose a concrete design.
- Socratic questions rather than handed-down solutions; engage with the
  question before producing code.
- Direct corrections when you overclaim or over-engineer; concede cleanly
  and adjust.

Do not pad responses with apologies, hedges, or restatements of the
task. Do not ask for clarification unless the task genuinely cannot
proceed without it; prefer to make a judgment call, state the assumption
clearly, and proceed.

When the maintainer pushes back on a design choice, the default
assumption is that the maintainer is right. The codebase has a strong
opinion; learn the opinion before fighting it.

## What a good change looks like

A good change in this codebase:

- Is small. One concept per change.
- Touches the spec only when it has to, and explicitly when it does.
- Matches the existing code style in the file being edited.
- Adds no new dependency without explicit discussion.
- Does not "improve" unrelated code "while in there."
- Has a commit message that explains *why*, not just *what*.
- Leaves the build green and the tests passing.

A change that doesn't fit this shape isn't necessarily wrong, but it
needs justification.

## What a bad change looks like

These are real failure patterns AI agents have hit on this codebase:

- **The "just adding a small abstraction" change.** A new wrapper class
  appears that has one caller, one method, and no clear future. Delete
  the wrapper, inline the call.
- **The "while I was in there" change.** A bug fix or feature also
  reformats fifteen unrelated functions. The reformat is reverted; the
  feature lands separately.
- **The "AI-shaped C++" change.** Five `std::optional`s, three
  `std::shared_ptr`s, a factory class, and an `enum class` for what should
  be three named constants. Rewritten in plain C-with-classes style.
- **The "speculative protocol field" change.** A new field is added to
  the wire protocol "in case we need it later" with no current consumer.
  Removed; added back when a consumer actually appears.
- **The silent spec change.** An engine event's parameter list is
  changed to add a new key; `PROTOCOL.md` is not updated. Reverted; the
  spec is updated first, then the engine.

If you find yourself making any of these, stop and reconsider.

## Working across the spec/implementation boundary

When a task touches both the spec and an implementation:

1. Understand the spec change first. What is the new event, command, or
   field? What semantics does it have? How does it interact with existing
   parts of the spec?
2. Update `PROTOCOL.md` first. The spec is the source of truth.
3. Implement the engine side. Verify it emits the new wire format
   correctly.
4. Implement the frontend side. Verify it parses and renders the new
   wire format correctly.
5. Test end-to-end with NetHack actually running. The conformance test
   for v1.0 is "NetHack plays correctly."

Do not reorder these steps. Implementation-first changes drift away from
the spec and accumulate quietly until the next person tries to write a
different implementation and discovers the divergence.
