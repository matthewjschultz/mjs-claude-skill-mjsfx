# mjsfx-sound-design — Claude Code skill

A Claude Code skill that makes Claude genuinely good at designing game sound effects with
[MJSFX](https://mjsfx.app) through its MCP tools — both reproducing the classic SFX canon
(coins, lasers, explosions, jumps, power-ups, UI blips, cohesive families) and inventing
original, experimental sounds and textures.

## Requires

The **MJSFX MCP tools**, exposed by MJSFX (Direct-download build) via **Settings ▸ Connect to AI**.
The app ships a headless `mjsfx-mcp` helper; the panel gives you a ready-to-paste config for Claude
Desktop or Claude Code. See the [MJSFX MCP user guide](https://mjsfx.app).

## Install

Via the marketplace (`matthewjschultz/mjs-claude-skills`):

```
/plugin marketplace add matthewjschultz/mjs-claude-skills
/plugin install mjsfx-sound-design
```

## What it does

- The **generate → audition → tweak → layer → effect → save** loop, and which MCP tool does each step.
- **Character recipes** — the one or two params that own each sound's identity (coin = the arp, laser =
  the downward sweep, explosion = noise + long decay + closing filter, …).
- **Multi-voice layering** for composite sounds.
- **Inventing new sounds** — exploration technique (roll wide and curate, push past preset ranges, the
  extended palette, layering unrelated voices, FX as transformation, design-from-a-feeling).

Canonical source lives with the app's MCP server in the MJSFX repo; this repo is the published plugin.
