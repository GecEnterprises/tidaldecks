# Semantic model

Status: accepted direction; not implemented in full.

## Atomic but composable

One evaluated deck expression is validated as a unit before TidalDecks sends or
installs any part of it. Unmentioned fields are unchanged rather than reset.

```haskell
d1 $
  setsong "funstu" $
  play $
  jumpcue 2
    # volume 0.5
```

Composition has three categories:

| Category | Examples | Composition rule |
| --- | --- | --- |
| Desired state | `setsong`, `play`, `pause`, `volume 0.5` | Last specified value wins; reapplying the effective value is a no-op. |
| Immediate action | `jumpcue`, a future beat jump | Actions retain an explicit execution order. |
| Installed behavior | `tracktrigs`, parameter automation | Installation replaces the previous behavior in its ownership scope. |

Atomicity is an application boundary, not database-style rollback. A defined
soft failure, such as jumping to a missing hotcue, may be logged and skipped
without undoing the rest of the plan.

## Loading is a barrier

When a plan changes the loaded track, the operations are applied in this order:

```text
resolve Load Code
-> load the track if it differs
-> wait for the matching load generation
-> apply desired state and immediate actions
-> install track-scoped behavior
```

Requests for the same pending track merge into the pending plan. Reasserting the
already-loaded Load Code does not reload the deck and does not clear installed
behavior.

## Track generations

Installed behavior belongs to a deck and a specific loaded-track generation,
not just to a filename or Load Code. Loading a different track advances the
generation and clears behavior owned by the previous generation. Stale load and
scheduler callbacks must carry a generation token and be ignored.

## Cross-deck actions

An installed behavior may be owned by one deck and target another:

```haskell
d2 $ tracktrigs (
  onbeat 64 (d1 $ jumpcue 3)
)
```

Deck 2 owns the trigger lifecycle. The action produced when it fires is a new,
validated request targeting deck 1.

## Setlists

A setlist is a declarative chain of reusable track plans and handoff plans. It
is compiled into an execution graph rather than flattened into a FIFO command
queue, because preparing and transitioning between decks requires concurrent
operations and explicit dependencies.

The same plans remain usable outside a setlist:

```haskell
d2 $ funst1

d1 $ into (d2 $ funst1) fadeAt64
```

See [Setlists and handoffs](setlists.md) for the syntax, execution model, and
reachability diagnostics.
