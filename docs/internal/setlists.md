# Setlists and handoffs

Status: accepted architectural direction and proposed syntax; not implemented.

A setlist describes an ordered DJ performance whose next track may be prepared
and mixed before the current track finishes. It is not TidalCycles `arrange`, a
playlist, or a FIFO list of OSC commands.

## Reusable plans

Track plans and handoff plans are independent values:

```haskell
funstu =
  playtrack "funstu"

funst1 =
  playtrack "funst1"
    # volume 1

fadeAt64 =
  atbeat 64 $
    smartfade 4
      # outgoing (
          lowcut (ramp 0.5 0 (overbeats 4))
        )
```

`playtrack` is convenience for a track plan that asserts the requested song and
playing state. It does not force unrelated parameters back to defaults.

A setlist chains the reusable values:

```haskell
decksAlternating =
  setlist "Decks Alternating" $
  alternating d1 d2 $ do
    item funstu
    handoff fadeAt64

    item funst1
    handoff $
      atend $
        smartfade 8

    item anotherTrack
```

The same track plan remains atomic outside the setlist:

```haskell
d2 $ funst1
```

The same handoff can be installed between the currently loaded outgoing deck
and an explicit incoming request:

```haskell
d1 $ into (d2 $ funst1) fadeAt64
```

An immediate ad-hoc transition is:

```haskell
d1 $ into (d2 $ funst1) $
  smartfade 4
```

The outer deck is outgoing. The nested deck request supplies the incoming deck
and track. The `HandoffPlan` is therefore the reusable unit shared by setlists
and smaller live-coded transitions.

## Conceptual types

```haskell
data TrackPlan = TrackPlan
  { loadCode :: LoadCode
  , initialState :: DeckPlan
  , installedBehavior :: InstalledBehavior
  }

data HandoffPlan = HandoffPlan
  { condition :: HandoffCondition
  , transition :: TransitionPlan
  }

data SetItem = SetItem
  { trackPlan :: TrackPlan
  , handoffToNext :: Maybe HandoffPlan
  }

data SetPlan = SetPlan
  { setName :: Text
  , deckPolicy :: DeckPolicy
  , setItems :: [SetItem]
  }
```

The concrete representation may change, but these semantic boundaries should
remain. `alternating d1 d2` is a deck-assignment policy; it does not change the
track or handoff values stored inside the list.

## Execution graph

Compilation turns a `SetPlan` into a graph rather than a command queue:

```text
prepare funstu on d1
-> play funstu
-> prepare funst1 on d2
-> wait for d1 track beat 64
-> start smartfade
     |-> start d2
     |-> automate d1 lowcut
     |-> automate the mix
     `-> maintain the BPM and phase relationship
-> stop d1
-> make funst1 current
-> prepare the following item on d1
```

Graph nodes carry dependencies, ownership, completion criteria, and a track
generation. A re-evaluated setlist may supersede nodes that have not started;
stale callbacks must not complete nodes belonging to an older set run.

## Smart fade contract

`smartfade beats` initially means:

1. Resolve and preload the incoming track on its assigned deck.
2. Verify that source and target provide usable BPM and beatgrid information.
3. Tempo-match and phase-align the incoming deck.
4. Start the incoming deck when the handoff condition becomes true.
5. Apply incoming, outgoing, and mix automations for the requested beat count.
6. Finish with the incoming deck fully active and stop the outgoing deck.

The transition captures its own beat clock at the instant it starts, so its
duration does not become ambiguous when the master deck changes.

Missing BPM or beatgrid information makes `smartfade` unavailable. A fallback
must be explicit:

```haskell
smartfade 4 # fallback (fade 4)

smartfade 4 # fallback cut
```

Without a fallback, preflight reports an error rather than silently changing
the transition style.

## Analysis and liveness

Every compiled node exposes a state:

```haskell
data NodeStatus
  = Ready
  | Running
  | Waiting WaitReason
  | Blocked BlockReason
  | Impossible Diagnostic
  | Completed
  | Superseded
```

`checkset` performs static analysis from the set plan and metadata, then may add
dynamic observations from Mixxx:

```haskell
checkset decksAlternating
```

Example report:

```text
Setlist: Decks Alternating

ok  funstu and funst1 resolve in the library
ok  both tracks provide BPM and beatgrid information

blocked  handoff from funstu waits for track beat 64
         the current loop 2-4 prevents progress beyond beat 4

waiting  funst1 depends on the blocked handoff
```

Diagnostics distinguish:

- **Impossible:** immutable facts make completion impossible, such as waiting
  for beat 96 on a track that ends at beat 80.
- **Contradictory:** the set plan itself installs loop 2-4 while requiring beat
  96 on the same traversal.
- **Blocked:** mutable external state, such as a manually enabled loop, prevents
  current progress but could be changed.
- **Waiting:** normal progress toward a reachable condition or dependency.
- **Unknown:** required state or metadata has not arrived.
- **Missed:** a crossing-only deadline has already passed on this traversal.

A condition must provide both runtime evaluation and an explanation of why it
is waiting. The scheduler must never reduce all non-progress to a generic
timeout.

## Runtime ownership

The TidalDecks runtime owns the full `SetPlan`, execution graph, reachability
analysis, preloading policy, and transition semantics. The Mixxx adapter owns
only bounded deck requests and short-lived load barriers.

The current Mixxx adapter's `m_pendingByDeck` contains at most one coalesced load
request per deck. It is not the future setlist queue and should not become one.
