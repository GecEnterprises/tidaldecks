# Runtime and protocol boundaries

Status: architectural constraint.

## Ownership

TidalDecks owns:

- expression validation and composition;
- trigger and automation definitions;
- deck and track-generation state;
- beat-clock prediction and resynchronization;
- scheduler ownership and conflict resolution;
- setlist compilation, reachability analysis, and transition policy;
- logging invalid expressions and runtime installations.

The upstreamable Mixxx OSC controller owns:

- generic versioned control reads, writes, triggers, and subscriptions;
- safe thread handoff to `ControlProxy`;
- generic feedback needed to observe transport and controls;
- no knowledge of `tracktrigs`, `ramp`, `fadein`, or TidalDecks syntax.

The downstream Mixxx TidalDecks adapter may prototype capabilities that require
library Load Codes or atomic load barriers, but those concepts must not leak
into the generic OSC protocol without a generic use case.

## Mixxx diagnostics UI

Mixxx may expose a read-only, skin-integrated TidalDecks diagnostics panel next
to the library. Its initial content is adapter-owned lifecycle information:
per-deck request/load state and a bounded, deck-labelled command history.

The panel must distinguish adapter observations from TidalDecks runtime state.
Until the runtime publishes versioned status snapshots to the downstream
adapter, runtime status is explicitly unavailable rather than inferred from
Mixxx controls. The panel does not create scheduling, setlist, trigger, or
automation controls in Mixxx.

## Scheduler

The runtime should use one scheduler loop, not one thread per trigger or
automation. It evaluates all active behavior, coalesces writes by deck/control,
and sends an exact final automation value even if intermediate UDP updates are
dropped.

Initial scheduling is control-rate and not sample-accurate. OSC timetags do not
change that. A later Mixxx-side musical scheduler may accept timestamped batches,
but it is separate from basic OSC control.

## Required feedback

Runtime triggers and automations depend on work intentionally deferred from the
first OSC phase:

- control subscriptions and feedback;
- loaded-track identity or generation notifications;
- track load completion or request acknowledgement;
- play state, rate, beat position, and usable beat-clock information;
- reconnect and state-resynchronization rules.

The runtime must not install track-scoped behavior until it can associate the
installation with a confirmed track generation.

## Setlist support

The Mixxx adapter's pending-deck map is a load coalescer, not a setlist queue.
TidalDecks keeps the complete execution graph and sends only bounded requests to
Mixxx.

Setlist execution additionally requires:

- request identifiers and structured accepted/loading/applied/rejected replies;
- a monotonically changing track generation for every deck;
- confirmed preload completion without starting transport;
- safe Load Code resolution and explicit metadata for BPM, beatgrid
  availability, duration, and reachable final beat;
- observable loop boundaries, sync state, rate, and current track beat;
- cancellation or supersession tokens for stale set runs.

The generic OSC protocol should provide generic subscriptions and control
feedback. Library resolution, track generations, and load barriers may remain
in the downstream TidalDecks adapter until a generic upstream use case exists.

`smartfade` remains TidalDecks policy and must not become a generic Mixxx OSC
operation. Initially the runtime expands it into deck requests and generic
control updates. A future Mixxx-side scheduler, if measurements justify one,
should accept bounded generic timestamped batches rather than an entire
`SetPlan`.
