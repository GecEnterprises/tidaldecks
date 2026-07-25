# Implementation roadmap

Status: working implementation plan; every phase ends with automated checks and
manual validation in Mixxx and Pulsar.

This roadmap covers the complete direction currently described by the internal
design notes: atomic controls, observable runtime state, automations, track
triggers, reusable handoffs, and executable setlists. A later phase may refine
an earlier API, but it must preserve the semantic categories of desired state,
ordered action, and installed behavior.

## Repository boundaries

Every phase must keep these ownership rules intact:

- the upstreamable Mixxx OSC controller provides generic versioned control
  reads, writes, triggers, normalized parameter access, and subscriptions;
- the downstream Mixxx TidalDecks adapter provides Load Codes, track-generation
  acknowledgements, and bounded atomic deck requests;
- the TidalDecks monorepo provides the Haskell API and runtime, Pulsar package,
  examples, tests, and documentation;
- scheduling policy, automations, triggers, handoffs, and setlists never become
  special operations in the generic Mixxx OSC protocol.

## Completed baseline

The current baseline supports Load Codes, idempotent track loading, play,
pause, stop, hotcue jumps, volume, composition of the fixed deck plan, Pulsar
line evaluation, visible evaluation errors, and Mixxx-side OSC logging.

Before starting a phase, the previous phase must be committed and manually
accepted. A failed manual checkpoint is fixed in the current phase rather than
carried as assumed future work.

## Phase 1: typed intent and extensible adapter protocol

Replace the growing fixed collection of optional fields with explicit desired
state and ordered action operations. Desired state composes last-value-wins by
ownership key; actions retain source order. Installed behavior gets a distinct
representation even though the scheduler does not execute it yet.

Replace the fixed TidalDecks OSC deck packet with a versioned representation
that can carry a variable number of typed operations. The adapter validates the
whole request before applying it. Add request IDs and structured
accepted/applied/rejected replies without adding TidalDecks concepts to the
generic OSC surface.

Preserve all existing source syntax and behavior.

Validation:

- unit tests for right-biased state, ordered actions, and full-request rejection;
- round-trip and malformed-packet tests;
- repeat load, transport, cue, and volume examples in Pulsar;
- repeat a desired state and confirm no duplicate load or toggle occurs;
- inspect the Mixxx OSC log for one lifecycle per request.

## Phase 2: looping

Add deck primitives:

```haskell
d1 $ loopbeats 4
d1 $ loop on
d1 $ loop off
d1 $ loopin
d1 $ loopout
d1 $ reloop
d1 $ loophalve
d1 $ loopdouble
```

`loopbeats n` ensures an `n`-beat loop is active and never toggles a matching
active loop off. `loop on` activates an existing valid loop without jumping;
`loop off` exits it. The remaining forms are ordered actions.

Relevant Mixxx controls include `beatloop_size`, `beatloop_activate`,
`loop_enabled`, `loop_in`, `loop_out`, `reloop_toggle`, `loop_halve`, and
`loop_double`.

Validation:

- create and exit 1-, 4-, and 8-beat loops while playing;
- repeat `loopbeats` and confirm the loop remains active;
- pause inside a loop and confirm loop state is preserved;
- validate loop-in/out, halve, double, and reloop on the waveform.

## Phase 3: tempo and sync

Add deck primitives:

```haskell
d1 $ bpm 128
d1 $ speed 1.05
d1 $ sync on
d1 $ sync off
d1 $ sync leader
d1 $ beatsync
d1 $ phasesync
d1 $ temposync
```

`bpm` expresses desired effective playback BPM. `speed` is a playback ratio
where `1.0` is analyzed speed; the API does not expose Mixxx's pitch-range-
dependent raw `rate` slider. Persistent sync and one-shot synchronization stay
distinct.

Relevant Mixxx controls include `bpm`, `file_bpm`, `rate_ratio`,
`sync_enabled`, `sync_leader`, `beatsync`, `beatsync_phase`, and
`beatsync_tempo`.

Validation:

- set and repeat BPM on an unsynced deck;
- validate `speed` under two configured pitch ranges;
- validate tempo-only, phase-only, one-shot beat sync, follower sync, and
  leader selection with deliberately mismatched decks;
- confirm stale operations cannot cross a track load generation.

## Phase 4: channel EQ and normalized parameters

Add deck primitives:

```haskell
d1 $ low 0.5 # mid 0.5 # high 0.5
d1 $ lowkill on # midkill off # highkill on
```

Values use normalized parameter semantics. Extend the generic Mixxx OSC
surface with a normalized parameter read/write operation corresponding to
Mixxx `getParameter` and `setParameter`; keep the friendly EQ names exclusively
in TidalDecks. Mixxx deck aliases include `filterLow`, `filterMid`,
`filterHigh`, and their `*Kill` controls. Reserve `lowcut` and `highcut` for
actual frequency-cut filters.

Validation:

- move each EQ band through minimum, neutral, and maximum;
- validate kills independently and in composition;
- switch the configured Mixxx equalizer and confirm normalized values remain
  useful;
- confirm a UI move remains untouched until another expression is evaluated.

## Phase 5: mixer and crossfader

Introduce a mixer target and deck crossfader assignment:

```haskell
mixer $ crossfader (-1)
mixer $ crossfader 0
mixer $ crossfader 1

d1 $ crossfade left
d2 $ crossfade right
```

The global control is `[Master], crossfader`, range `-1 .. 1`; each deck uses
`orientation`. `MixerPlan` follows the same desired-state composition rules as
`DeckPlan`.

Validation:

- assign decks left, right, and center/bypass;
- move the crossfader to both edges and center;
- repeat positions and confirm no duplicate behavior;
- confirm deck volume and mixer state do not overwrite each other.

## Phase 6: effect routing and shared units

Keep deck routing distinct from shared effect-unit state:

```haskell
d1 $ fxsend 1 on # fxsend 2 off
fx1 $ enabled on # wet 0.4 # super 0.7
fx2 $ enabled off
```

Deck routing maps to controls such as
`[EffectRack1_EffectUnit1], group_[Channel1]_enable`. Shared unit state uses
`enabled`, `mix`, and `super1`. Effect selection and effect-specific slot
parameters remain deferred until their identity and discovery model is designed.

Validation:

- route different decks to FX1 and FX2;
- confirm shared-unit changes affect all and only routed sources;
- validate unit enable, dry/wet, and super-knob endpoints;
- disable a route without resetting the shared unit;
- repeat desired states and confirm no toggle behavior.

## Phase 7: generic feedback and runtime state

Complete generic OSC `get`, `subscribe`, `unsubscribe`, and value feedback.
Add normalized parameter feedback. Keep networking and parsing off the audio
thread, bound subscription and outbound queues, and preserve loopback-only and
runtime-enable defaults.

The adapter adds structured load lifecycle replies, monotonically changing
track generations, resolved Load Code, loaded-track metadata, beatgrid
availability, duration, current track beat, loop boundaries, and cancellation
or supersession tokens. Use generic controls wherever they already express the
state; keep library-specific identity in the adapter.

The Haskell runtime gains a state cache, subscription ownership, timeout and
reconnect handling, and full resynchronization after either process restarts.

Validation:

- observe play, BPM, rate, volume, EQ, loop, sync, and beat-position changes
  made both through TidalDecks and manually in Mixxx;
- load several tracks rapidly and reject stale acknowledgements;
- restart Mixxx, then restart TidalDecks, and verify reconstructed state;
- measure feedback loss and queue behavior under a deliberately high update rate;
- display connection and request lifecycle errors clearly in Pulsar.

## Phase 8: automation scheduler

Implement one control-rate scheduler loop and parameter ownership. Begin with
volume, then allow every safe normalized desired-state parameter to use the same
automation value type:

```haskell
d1 $ volume (fadein (overbeats 2))
d1 $ volume (ramp 0 1 (trackbeats 42 48))
d1 $ low (ramp 0.5 0 (trackbeats 64 68))
```

`overbeats` advances only during playback and ignores seeks. `trackbeats`
follows track position and repeats through loops. A constant assignment cancels
automation for that ownership key. Track-scoped automation is cleared by a new
track generation. The scheduler coalesces writes and always sends the exact
final value.

Validation:

- pause and resume a relative fade;
- seek and loop through a track-beat ramp;
- replace an automation, cancel it with a constant, and run independent
  automations together;
- load a different track and verify old automation cannot write again;
- measure update rate, audible stepping, CPU use, and OSC traffic.

## Phase 9: track triggers

Implement track-generation-scoped discrete triggers:

```haskell
d2 $ tracktrigs (
  onbeat 64 (d1 $ jumpcue 3)
)
```

`onbeat` fires on forward playback crossing, not floating-point equality or a
manual seek. Returning below the beat rearms it. Replacing `tracktrigs`
replaces the complete set owned by that source deck generation. Trigger bodies
are validated when installed and guarded again when fired.

Validation:

- cross the target normally, by seek, and through a loop;
- replace and clear trigger sets;
- load a new source track and verify old triggers disappear;
- fire deck-local, cross-deck, and automation-producing actions;
- confirm removing a source trigger does not cancel an automation already
  launched on its target deck.

## Phase 10: reusable track plans and handoffs

Implement reusable `TrackPlan` and `HandoffPlan` values independently of full
setlists:

```haskell
funst1 = playtrack "funst1" # volume 1

fadeAt64 =
  atbeat 64 $
    smartfade 4
      # outgoing (low (ramp 0.5 0 (overbeats 4)))

d1 $ into (d2 $ funst1) fadeAt64
```

Implement preload without transport, handoff conditions, transition clock
capture, BPM and phase preparation, incoming/outgoing/mixer automations, final
state, and explicit fallback policy. `smartfade` is compiled by TidalDecks and
never added to the generic OSC protocol.

Validation:

- use the same track plan alone and as an incoming plan;
- use the same handoff immediately and at a track beat;
- validate alternating outgoing decks;
- test missing BPM/beatgrid with no fallback, `fallback (fade n)`, and
  `fallback cut`;
- supersede a prepared incoming track and reject stale completion;
- confirm the completed handoff stops the outgoing deck and leaves the incoming
  deck fully active.

## Phase 11: setlist compiler and diagnostics

Implement declarative setlists and compile them into an execution graph:

```haskell
decksAlternating =
  setlist "Decks Alternating" $
  alternating d1 d2 $ do
    item funstu
    handoff fadeAt64
    item funst1
    handoff $ atend $ smartfade 8
    item anotherTrack
```

Graph nodes carry dependencies, ownership, completion criteria, set-run ID,
and track generation. Implement `checkset` with static metadata analysis and
dynamic Mixxx observations. Diagnostics distinguish ready, running, waiting,
blocked, impossible, contradictory, missed, completed, and superseded states.

Validation:

- inspect a valid three-track graph before playback;
- detect a handoff beyond track duration and a planned loop contradiction;
- report a manually enabled blocking loop and resume when it is removed;
- report missing tracks, metadata, beatgrids, and unavailable decks;
- render useful `checkset` output in Pulsar without starting playback.

## Phase 12: setlist execution and live replacement

Execute the compiled graph with alternating deck preparation, bounded adapter
requests, transition scheduling, completion propagation, and liveness
diagnostics. The adapter's pending-deck map remains only a short-lived load
coalescer; it never becomes the setlist queue.

Define and implement re-evaluation semantics: a new evaluation supersedes
unstarted nodes from the old set run, generation and set-run tokens guard all
in-flight callbacks, and already-running transitions reach an explicitly
defined safe boundary before replacement.

Validation:

- run a multi-track alternating setlist end to end;
- edit future items while the current item plays;
- replace the set during preload and during a handoff;
- restart or disconnect either process and report recoverable versus abandoned
  execution accurately;
- exercise blocked, missed, superseded, and fallback paths without indefinite
  silent waiting;
- verify the complete performance can still be interrupted with atomic manual
  deck expressions.

## Phase 13: public API and first-runtime review

Review the full vocabulary and decide whether beat jump, key lock/key shift,
headphone cue, pregain, samplers, quick effects, or effect-slot discovery are
required before the first public runtime. Resolve or explicitly defer every
item in `open-questions.md`, finish the conflict matrix, publish protocol and
reconnect guarantees, add examples, and make Pulsar completion/highlighting
match the actual API.

Run a long-form manual DJ session and capture performance measurements before
declaring the first runtime stable. Sample-accurate or Mixxx-side musical
scheduling remains a separate later project justified by measurement, not an
assumption built into basic OSC control.
