# TidalDecks workspace instructions

This is the coordination repository for the TidalDecks system. Read
`docs/internal/implementation-phases.md` first for progression work and keep
cross-repository design decisions in `docs/internal/`.

## Repository layout

- `repos/tidaldecks-lib/`: Haskell API/runtime, Pulsar package, examples, and
  package-local documentation.
- `repos/mixxx/`: Mixxx fork and the downstream TidalDecks adapter.
- `docs/internal/`: roadmap, semantics, runtime boundaries, and open questions
  shared by both implementations.
- `tools/`: commands that operate on the complete workspace.

## Change workflow

1. Inspect `git -C repos/tidaldecks-lib status -sb` and
   `git -C repos/mixxx status -sb` before editing.
2. Make and test changes in the owning submodule.
3. Commit and push each changed submodule first.
4. Update and commit the submodule pointers and any root documentation here.

Do not add TidalDecks scheduling or musical policy to generic Mixxx OSC. Keep
the ownership boundaries in `docs/internal/runtime-boundaries.md` intact.
