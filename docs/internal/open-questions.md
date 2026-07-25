# Open questions

These questions must be decided or tested before implementing the runtime:

1. Are public track beats one-based, and how do they map to Mixxx beat-grid
   coordinates before the first beat?
2. How does the runtime distinguish a manual seek from ordinary playback when
   evaluating `onbeat` crossings?
3. What feedback rate is sufficient for audible volume automation, and which
   Mixxx controls already smooth incoming changes?
4. Does editing or replacing a beatgrid invalidate installed behavior, or should
   it immediately follow the revised grid?
5. Should an explicit UI parameter change cancel automation in a future opt-in
   mode, even though the initial rule keeps runtime ownership?
6. What acknowledgement identifies the exact track generation created by an
   atomic load request?
7. How are runtime state and ownership reconstructed after either Mixxx or
   TidalDecks reconnects?
8. Should multiple immediate actions preserve source order, or should some
   action types have named phases around transport changes?
9. Which errors reject an entire intent, and which are defined soft failures
   like a missing hotcue?
10. How should multi-deck actions express a common musical deadline without
    implying sample-accurate simultaneity?
11. Is `atbeat` crossing-only, and what explicit operator requests the next
    traversal after a missed beat?
12. How early should a setlist preload its next item, and what happens when no
    idle deck is available?
13. Which controls and EQ moves are part of the default `smartfade`, versus
    explicit `incoming` and `outgoing` modifiers?
14. Which deck supplies the captured transition beat clock while BPM matching
    and master ownership change?
15. May a running setlist be edited structurally, or must an evaluation replace
    every unstarted node after a stable handoff boundary?
16. What is the default behavior when a mutable blocker, such as a manual loop,
    persists indefinitely: wait, warn repeatedly, or apply a declared recovery
    policy?
