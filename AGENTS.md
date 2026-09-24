# protohack — spec repo

This repo contains the Protohack protocol specification and design
documentation. It has no engine code and no frontend code. Engines and
frontends are sibling repos in the workspace.

## Contents

```
protohack/
├── docs/
│   ├── DESIGN.md          — architectural philosophy (the why)
│   ├── ARCHITECTURE.md    — reference implementation structure
│   ├── AGENTS.md          — coding discipline for AI and human contributors
│   └── PROTOCOL.md        — wire format and event vocabulary (the spec itself, TBD)
└── CLAUDE.md              — this file
```

## Key invariants

- **The spec is the canonical artifact.** `docs/PROTOCOL.md` is the project.
  All engines and frontends are reference implementations or conformance
  tests. Spec changes are the most consequential changes in the ecosystem.
- **No engine or frontend code lives here.** If you find yourself writing
  game logic, subprocess management, or rendering code in this repo, stop.
- **Conformance test:** a Protohack-conformant engine+frontend pair must be
  able to play NetHack to completion. If a spec change breaks that, the
  spec change is wrong.

## Sibling repos

| Repo | Role |
|------|------|
| `../hack-client/` | Python frontends (pygame + Textual). Reference frontend implementations. |
| `../nethack-protohack/` | NetHack fork with `hack2bridge` windowport. Reference engine implementation. |
| `../godot-frontend/` | Godot frontend. |
| `../gamemaker-frontend/` | GameMaker frontend. |

Read `docs/DESIGN.md` before making any changes to the spec.
