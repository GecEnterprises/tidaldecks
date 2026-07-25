# Parameter automations

Status: accepted syntax direction; runtime not implemented.

Automations are installed behavior that continuously owns one parameter. The
public vocabulary is intentionally DJ-oriented and hides scheduler sampling,
OSC update rates, and control-bus concepts.

## Syntax

Constants retain the compact form:

```haskell
d1 $ volume 0.5
```

Relative and track-position automations use nested constructors:

```haskell
d1 $ volume (fadein (overbeats 2))

d1 $ volume (fadeout (trackbeats 96 100))

d1 $ volume (ramp 0.25 0.8 (overbeats 4))

d1 $ volume (ramp 0 1 (trackbeats 42 48))
```

The conceptual API is:

```haskell
overbeats  :: BeatCount -> Window
trackbeats :: Beat -> Beat -> Window

ramp   :: Double -> Double -> Window -> Automation
fadein :: Window -> Automation
fadeout :: Window -> Automation
```

`fadein` is `ramp 0 1`; `fadeout` is `ramp 1 0`. Future curves such as
`easein` and `easeout` should keep the same final `Window` argument.

## Clock domains

`overbeats n` is relative musical duration on the target deck. It begins when
the plan is applied, advances only while the deck plays, pauses with the deck,
and is not moved forward or backward by cue jumps or seeking. On completion it
sets the exact final value and releases ownership, leaving that value in place.

`trackbeats start end` is a function of the loaded track's beat coordinate. It
holds the start value before the range, interpolates within the range, and holds
the end value after it. It remains installed for the track generation, so
looping or seeking back through the range repeats the envelope.

Both forms require a usable beat clock. Missing or invalid beat information is a
runtime error: log and drop the installation without changing the parameter.

## Ownership

- There may be one active automation per deck parameter.
- A new automation for that parameter atomically replaces the old one.
- A constant assignment cancels the automation and sets the constant.
- Automations on different parameters coexist.
- Loading a different track clears track-scoped automations.
- Reasserting the currently loaded track preserves them.
- While an automation owns a parameter, a manual UI change is temporary; the
  next runtime update restores the computed value.

Values are interpolated in Mixxx control space. Whether a control is perceptually
linear is a separate concern that can be expressed by future curve types.
