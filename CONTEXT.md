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

**Prefer lazy computation over eager, especially anything gated by
`objects:`/`poke:` nearby scans.** A scan should only run the first time its
result is actually read, cached after that — not run up front for every
rule that merely declares `objects:`/`poke:`, whether or not a script reads
`<objectcount>`/`<pokecount_X>` at all. Confirmed by Jere's own follow-up
commit to our `<objectcount>` PR (`3f29f88`, 2026-09-29): our version
computed the full nearby-object count in `PrefabManager.Handle` before the
rule's own templates ever resolved; his rewrite deferred the same scan into
a `objectCounts ??= objects == null ? [] : ObjectsFiltering.GetCounts(...)`
getter, computed once on first read and cached, never touched if the script
never asks. He applied the identical shape to a brand-new `<pokecount_X>`
feature in the same commit. When adding a new function that depends on a
nearby-object/poke-target scan, default to this lazy-getter shape rather
than an eager pre-computation, unless there's a specific reason the result
is needed unconditionally.

**This is about avoiding speculative O(n) work, not banning all per-call
cost.** A cheap O(1) check that runs on every call (a single dictionary
lookup, for example) is a different order of magnitude from an eager scan
over every nearby object, and doesn't meaningfully violate the principle
above. Confirmed while designing an RPC-failure diagnostic (2026-09-29,
`.scratch/rpc-name-audit`): the human initially read the objectcount lesson
as "avoid any per-call cost at all" and considered a more fragile
zero-cost transpiler over a simple Harmony Postfix doing one
`Dictionary.ContainsKey` per call — settled on the Postfix once the actual
cost was named plainly. Don't over-apply "lazy" into rejecting trivially
cheap always-on checks.
