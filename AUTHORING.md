# Writing a workflow plugin

A plugin is a git repository of **pure JSON** — the platform validates everything at
sync time and never executes anything from it. Operators add your repo (URL + pinned
commit) in the admin console; tenants then select your workflows.

## Layout

```
plugin.json                  {"name", "description", "workflows": ["mykey", ...]}
mykey/workflow.json          the workflow manifest (below)
mykey/schemas/*.json         entity schemas
mykey/processes/*.json       phase/turn process definitions
mykey/rule_systems/<k>/definition.json   deterministic resolution rules
mykey/tools/*.json           tool definitions
mykey/axes/*.json            behavior axis definitions
mykey/overlay/*.json         vocabulary overlay {"key", "labels": {label_key: text}}
mykey/seed/*.json            library knowledge seed content
smoke.json                   (recommended) the pack's own CI smoke fixture
```

Workflow keys are **globally unique across all plugin repositories** in a
deployment; a sync that would claim another repo's key fails with a clear error.

## workflow.json

```json
{
  "key": "mykey",
  "name": "My Workflow",
  "overlay_key": "mykey_v1",
  "persona_type_labels": {"supervisor": "Host", "participant": "Guest"},
  "label_overrides": {},
  "featured_process_keys": [],
  "capabilities": {"mcp_servers": []}
}
```

## Content kinds

- **schemas** — `{"key", "definition"}`; the definition carries JSON-Schema-style
  fields, invariants, `state_machines` (states with `label_key`s, `initial`,
  transitions with `trigger` and optional CEL `guard` over `fields.*`), and `views`.
  Two things about a schema are silent when wrong. A field you declare and name in no
  view group is stored, validated, mutable by tools, and **never shown** — the sheet
  simply does not have it. And a JSON object with the same key twice parses fine,
  keeping the last and dropping the first, so a label added under a key that already
  exists further down an overlay has no effect. The platform's pack tests check both
  across every shipped file; run them before you pin.
- **processes** — `{"key", "name", "definition"}`; the definition is the phase/turn
  DSL: phases with prompts, actor rotation, visibility, budgets, and `tools`.
  **A round is an actor entry, not a `max_turns` number.** An entry with
  `order: "declared"` (or `"initiative"`) resolves the roster once and walks it once;
  `max_turns` caps that walk, it does not repeat it. So `{"persona_type": "participant",
  "order": "declared", "max_turns": 8}` in front of three players is three turns, not
  eight — which is how a combat phase that meant to run three rounds ran one, with a
  single roll in it. Want three rounds? Write the entry three times. Interleaving a
  supervisor entry between them is what gives the referee a turn *between* rounds
  instead of only at the start. (`order: "reactive"` and `"free"` do re-pick every turn,
  so `max_turns` is a turn count there — at the cost of no guarantee that everyone is
  reached.)
  **A phase can require that something happened.** `requires` on a phase names counts it
  must reach before it may transition — `resolutions`, `messages`, `tool_calls`,
  `entity_changes`, `entities_touched` — counted from the phase's own entry, so an
  earlier beat's work cannot satisfy a later one. Without it a phase ends when its actor entries are used up, which
  says the turns were spent and nothing about whether the work happened: on a live
  eight-beat run one good encounter ran on while four beats advanced underneath it, and
  the beat meant to stage a different opponent was spent on more rounds of the previous
  one.

  ```json
  "requires": { "resolutions": 1, "on_unmet": "repeat", "max_repeats": 2 }
  ```

  **Pick the counter that can see the work.** `entity_changes` counts state-change rows,
  and *creating* a record writes none of them — so a beat whose whole job is to make
  something counts zero against it and is reported unmet while the things sit in the
  table. That is what `entities_touched` is for: entities of this session created or
  written in this phase. A beat that makes characters wants `entities_touched`; a beat
  that damages them wants `entity_changes`. Getting this wrong is the likeliest way to
  write a gate that never passes, and it cost a live run two wasted repeats of character
  creation while three finished sheets already existed.

  `on_unmet` is `repeat` (run the rotation again, up to `max_repeats`, then move on),
  `hold` (park for a person instead) or `warn` (move on and record it). Moving on is the
  default end state everywhere but `hold`, because a beat nobody can satisfy must not
  trap a session. Every decision writes a `phase_requirement` event with both numbers, so
  a beat let through thin says so in the transcript.

  Use it where a beat has a job: a fight that must roll something, a review that must
  file a verdict, a step that must change a record. Do not use it as a quality bar — it
  counts rows, it cannot read them, and a phase that rolls once to satisfy a rule is not
  what you wanted.

  **One prompt, every seat.** `prompt` is per phase, not per actor entry, so a
  paragraph beginning "Referee:" is read by the players too — and a model that reads
  an instruction will carry it out. A player spent his character-creation turn
  auditing another player's sheet against the rules, accurately, and never rolled his
  own, because the referee's paragraph told somebody to. If a phase addresses more
  than one role, say at the top that only the paragraph addressed to your seat is
  yours, and name what taking the other seat looks like.
- **rule_systems** — check types, outcome bands, modifiers; what makes a resolution
  *mean* something. The `rpg` workflow ships three to start from: `generic_d20`
  (target-number checks per ability), `coin_flip` (a `1d2` with two outcome bands),
  and `generic` (a permissive grammar with no bands at all, for when the answer
  wanted is a number and nothing else). A specific published ruleset is not one of
  them — that travels with the game that uses it, in a `.pyr` bundle.
- **tools** — input/output JSON Schemas plus `impl_ref` and `validation_ref` (a rule
  system key). `impl_ref` must name a **platform builtin** — plugins cannot ship
  code. There is one: `builtin:randomizer`, a seeded expression roller whose
  behaviour comes entirely from the rule system it resolves in. Do not ship a second
  tool for a second kind of randomness — a coin is a `1d2` grammar with two outcome
  bands, an ungraded number is a permissive grammar with none, and both are rule
  systems. A call may name one with its `rule_system` argument; without one it uses
  the tool's `validation_ref`, and failing that the workspace's default.
- **axes / overlay / seed** — behavior dials, display vocabulary, starter knowledge.

Everything is validated at sync through the same models the loader uses — schema
errors, malformed process DSL, or invalid tool definitions fail the sync with the
offending file named, before any tenant can select the workflow.

## Capabilities: giving personas tools

`capabilities.mcp_servers` lists MCP servers to enable on workspaces using your
workflow. Two kinds:

1. **The `resolution` preset** — the platform serves your pack's registered
   deterministic tools over MCP: server-side randomness, every outcome a persisted
   resolution record models can only *narrate around*, never invent.

   ```json
   {"key": "resolution", "url": "pyrrhula://resolution/{tenant_id}",
    "enabled_tools": ["randomizer"], "effectful_tools": []}
   ```

   `{tenant_id}` is substituted at registration. Tools may pass
   `actor_entity_id` + `machine_key` + `trigger` to drive a state machine from the
   outcome in the same call.

2. **External MCP servers you operate** — any `http(s)://` URL, reached by the
   platform's generic MCP streamable-HTTP client (JSON-RPC `initialize` →
   `tools/list` / `tools/call`; plain-JSON and SSE responses both work — FastMCP
   servers work out of the box). The workspace allowlist (`enabled_tools` /
   `effectful_tools`) is the egress control, and allowed tools are surfaced to
   personas as native in-turn tools automatically. **Declared servers are shown to
   the operator on the plugin panel** — adding a repository is also approving its
   endpoints, so declare honestly and minimally. Operators can also attach servers
   to a tenant directly from the admin console (no plugin needed); those grants are
   merged after your pack's declared servers and win key collisions.

## Work items are units of code, not units of thought

A phase that creates `work_item` entities and a phase that calls `delegate_work_item` are
describing the same pipeline: each item goes to a coding agent that checks out a branch,
**writes files**, commits, and opens a pull request. There is no path through that pipeline
that reports a finding without a diff.

So an item like *"Diagnose the failing tests and decide which side is wrong"* is not a
smaller version of a coding task, it is a different kind of thing, and the pipeline has
nowhere to put it: the agent produces commits that change no files, the pull request has
nothing in it, and the only honest disposition is to close it. This is easy to author by
accident, because splitting investigation from implementation is exactly what a careful
human lead does.

**Investigation belongs in a phase.** `swdev/processes/investigate_plan_implement_review_merge.json`
has an `investigate` beat for precisely this: agents read the repository graph, exchange
what they found in the transcript, and the lead plans against the result -- no delegation,
no branch, no pull request. Put the thinking there, and let every work item be something a
reviewer could read a diff of.

Two practical consequences when writing the `plan` phase's prompt:

- Say that each item must name the files to create or modify and what changes in each. An
  item that cannot name a file is usually an investigation in disguise.
- Say that the description has to stand alone. The coding agent sees the item, not the
  conversation that produced it.

## Testing your plugin

Point a local checkout at a dev platform without pushing:
`PYRRHULA_PLUGINS_LOCAL_<name>=/path/to/checkout` (see the platform's
`scripts/fetch_plugins.py`), or add the repo in the admin console with a branch
commit sha and re-sync as you iterate. Ship a `smoke.json` per workflow so the
platform's pack-matrix CI can boot an entity, run a resolution, and drive a
transition from your own fixture.

State machines are optional, and most content does not need one: a workflow whose
entities have no lifecycle worth tracking should declare none. Where they do earn their
place, the platform's `docs/entities-and-state-machines.md` is the reference — what a
machine can express, how guards and effects are evaluated, and how an entity's state
travels in a `.pyr`.
