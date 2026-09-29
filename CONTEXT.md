# Expand World Prefabs (EWP) — context

## Design philosophy

**Server-authoritative by design.** Jere built EWP so its logic runs on the
server only — a client can join without installing EWP at all and see no
difference. Confirmed in code: `InfoManager.Patch()` gates nearly every
Harmony patch behind `canPatch = !Helper.IsClient()`, and `ServerOwned.cs`
exists specifically to keep objects owned by the server instead of a client,
"to receive RPCs that vanilla only routes to the owning client." Any new
feature that only works by also requiring client-side installation is
against the grain of the mod — flag this to the human before proposing one.

**Cost review (lazy over eager, O(1) is not the enemy):** general principle
and the cheap-vs-speculative distinction now live in the shared
[mod-cleanup-and-cost-review.md](../../docs/agents/mod-cleanup-and-cost-review.md)
(shared with `valheim-modding`) — read that before adding a new function
that depends on a nearby-object/poke-target scan, or any patch with a
per-call cost question.

EWP-specific example that principle came from: Jere's own follow-up commit
to our `<objectcount>` PR (`3f29f88`, 2026-09-29) rewrote our eager
`PrefabManager.Handle` scan into a
`objectCounts ??= objects == null ? [] : ObjectsFiltering.GetCounts(...)`
getter, computed once on first read and cached — applied the identical
shape to a brand-new `<pokecount_X>` feature in the same commit. Default
new nearby-scan functions to this lazy-getter shape, unless there's a
specific reason the result is needed unconditionally.
