# Project workspace UI

Status: implemented development convenience.

## Purpose

The skin-integrated TidalDecks panel reduces setup work without moving musical
runtime policy into Mixxx. It reports whether GHC, Cabal, Pulsar, and the
TidalDecks Pulsar package are discoverable, and it keeps an editable list of the
ten most recently selected project directories in Mixxx settings.

Browsing, creating, and opening a project moves it to the top of the history.
The selected history entry is the target of **Open in Pulsar**.

## Generated project

**Create project** accepts only a new or empty directory and creates:

- `cabal.project`, with a pinned `source-repository-package` revision;
- `tidaldecks-session.cabal`, a small local package depending on `tidal-decks`;
- `src/Session.hs`, the Haskell session module;
- `live.tidaldecks`, with starter deck and mixer expressions;
- `README.md`, with the short evaluation workflow.

After scaffolding, Mixxx adds the directory to project history and asks Pulsar
to open both the project and `live.tidaldecks`. Mixxx does not install missing
tools or packages and does not overwrite a non-empty directory.

## Editor handoff

The Pulsar package recognizes both development checkouts containing
`tidal-decks.cabal` and generated projects containing
`tidaldecks-session.cabal`. It starts the Cabal REPL for the matching component;
evaluation and all TidalDecks runtime behavior remain editor/library-owned.
