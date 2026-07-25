# Track triggers

Status: proposed runtime feature.

`tracktrigs` installs discrete actions on the current track generation:

```haskell
d2 $ tracktrigs (
  onbeat 64 (d1 $ jumpcue 3)
)
```

The intended semantics are:

- the trigger observes deck 2's loaded track and beat position;
- crossing beat 64 during forward playback emits the deck 1 action;
- replacing `tracktrigs` replaces the complete trigger set on deck 2;
- loading a different track on deck 2 clears the set;
- reasserting the same Load Code without a load preserves the set;
- trigger bodies are validated when installed and again guarded when fired.

For the first implementation, `onbeat` should fire on a forward crossing rather
than exact floating-point equality:

```text
previousBeat < targetBeat && currentBeat >= targetBeat
```

It should not fire merely because the playhead was manually moved from before
to after the target. Returning below the beat rearms it, so a loop can trigger it
again. A future one-shot operator should have a distinct name rather than change
`onbeat` semantics.

If a trigger launches an automation on another deck, that automation becomes
owned by the target deck. Removing the original trigger afterward does not
cancel an automation that has already started.
