# TidalDecks workspace

This repository coordinates the TidalDecks live-coding system and its Mixxx
adapter. The implementation repositories are independent Git repositories,
checked out here as pinned submodules:

```text
repos/tidaldecks-lib/   Haskell API, runtime, Pulsar package, examples
repos/mixxx/            Mixxx fork with the TidalDecks adapter
docs/internal/          cross-repository roadmap and design decisions
tools/                  workspace checks and bootstrap helpers
```

Clone the complete workspace with:

```shell
git clone --recurse-submodules https://github.com/GecEnterprises/tidaldecks.git
cd tidaldecks
./tools/bootstrap
```

The root repository owns progression documents and integration policy. Each
submodule remains independently buildable and must be committed and pushed
before its updated pointer is committed here.

Useful commands:

```shell
./tools/status
./tools/test
```

The library package is currently still named `tidal-decks`; the repository is
named `tidaldecks-lib` to distinguish it from this coordinating workspace.
