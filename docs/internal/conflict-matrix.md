# Conflict matrix

Status: living design checklist.

| Existing behavior | New intent or event | Result |
| --- | --- | --- |
| Track X is loaded | `setsong X` | Do not reload; preserve installed behavior and apply other fields. |
| Track X is loading | another `setsong X` plan | Merge into the pending plan; do not start another load. |
| Track X is loading | `setsong Y` | Y becomes the desired load; stale X completion cannot apply its plan. |
| Any track generation | different track loads | Clear its triggers and track-scoped automations. |
| Volume automation | `volume 0.5` | Cancel automation and set `0.5`. |
| Volume automation A | volume automation B | Replace A atomically with B. |
| Volume automation | rate automation | Both continue because ownership keys differ. |
| Relative automation | deck pauses | Freeze elapsed beat duration. |
| Relative automation | cue jump or seek | Continue from elapsed played-beat time. |
| Track-beat automation | cue jump or seek | Recompute from the new track beat position. |
| Track-beat automation | loop returns before range | Envelope moves back and runs again with track position. |
| Active automation | manual UI parameter change | Runtime restores its value on the next update. |
| Trigger set A | `tracktrigs` set B | Replace the entire set A with B. |
| Trigger launches target-deck automation | source trigger is later removed | Launched automation continues under the target deck's ownership. |
| Plan references missing hotcue | plan applies | Log and skip the cue action; other valid operations remain applied. |
| Installation needs absent beatgrid | installation requested | Log and drop it without changing the owned parameter or trigger set. |
| Setlist handoff waits for beat 96 | track metadata ends at beat 80 | Mark the handoff impossible and all strict dependents unreachable. |
| Setlist handoff waits for beat 96 | plan installs loop 2-4 | Report a contradictory set plan before playback. |
| Setlist handoff waits for beat 96 | Mixxx UI currently loops 2-4 | Mark it dynamically blocked; disabling the loop may resume progress. |
| Crossing-only handoff at beat 64 | execution begins after beat 64 | Mark it missed unless its policy explicitly allows the next traversal. |
| `smartfade` requires beatgrids | either track lacks one | Reject preflight unless the handoff declares a fallback. |
| Incoming setlist item is known | preceding item begins | Prepare it on the assigned idle deck without starting playback. |
| Setlist is re-evaluated | old graph has pending nodes | Supersede unstarted old nodes; generation tokens guard in-flight work. |
| Trigger launched by set item | set advances to another track generation | Remove triggers owned by the old generation; already-launched target actions retain their own ownership. |

Every new primitive must add rows for interactions with load barriers, track
generation changes, existing ownership, transport discontinuities, and manual
UI changes.
