# Internal design notes

These documents define proposed TidalDecks semantics before they become public
API. They are reasoning tools, not promises that the described syntax is
implemented.

The guiding principle is **atomic but composable**: an evaluation describes one
validated intent, while its primitives can be combined without resetting
unmentioned deck state.

- [Semantic model](semantics.md)
- [Track triggers](track-triggers.md)
- [Parameter automations](automations.md)
- [Setlists and handoffs](setlists.md)
- [Conflict matrix](conflict-matrix.md)
- [Runtime and protocol boundaries](runtime-boundaries.md)
- [Implementation roadmap](implementation-phases.md)
- [Open questions](open-questions.md)

When a new feature is proposed, first classify it as desired state, an immediate
action, or installed behavior. Then add its interactions to the conflict matrix.
