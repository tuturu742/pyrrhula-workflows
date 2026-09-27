# pyrrhula-workflows

The default workflow plugin for [Pyrrhula](https://github.com/tuturu742/pyrrhula):
two specialized workflows as **pure declarative content** — no code, only JSON.

| workflow | what it is |
|---|---|
| `rpg` | Tabletop RPG campaigns: Game Master + Players, generic seeded resolution (dice, coins, raw numbers), character schemas with health/condition state machines. A specific ruleset travels with the game that uses it, in a `.pyr` bundle, rather than living here. |
| `swdev` | Software development: Lead + Engineers, work-item schemas and lifecycle FSM, delegation to coding agents on real repositories with CI and review loops. |

## Structure

```
plugin.json                 the plugin: name + workflow list
<workflow>/workflow.json    manifest: labels, vocabulary overlay, capabilities
<workflow>/schemas/         entity schemas (JSON Schema + FSM definitions)
<workflow>/processes/       phase/turn process definitions
<workflow>/rule_systems/    deterministic resolution rules (grammar, checks, outcomes)
<workflow>/tools/           tool definitions (validated, declarative)
<workflow>/axes/            behavioral axis definitions
<workflow>/overlay/         vocabulary overlay (display labels)
<workflow>/seed/            library knowledge seed content
```

Everything a workflow contributes is data validated against the platform's published
schemas: JSON Schema for entities, declarative finite-state machines, CEL expressions
for conditions. **Plugins can never ship executable code** — that's a platform
guarantee, not a convention.

## Using it

This repository is registered as the default plugin in every Pyrrhula deployment
(`deploy/plugins.json` pins a commit). Operators add further plugin repositories in
the admin console; each provides its own workflows under globally-unique keys.

To build your own workflow plugin, copy one of these directories as a starting point
— the platform's pack loader treats any conforming directory identically.

## Licence

These workflow packs are **MIT** ([LICENSE](LICENSE)). Copy one, rewrite it, ship the
result — a pack is content (schemas, flows, axes, tool definitions and seed knowledge, all
JSON), not code you extend, so nothing here attaches an obligation to the workflow you
build from it.


Pyrrhula itself, the engine that loads these, is
[MIT](https://github.com/tuturu742/pyrrhula) as well.
