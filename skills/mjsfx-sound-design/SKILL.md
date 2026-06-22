---
name: mjsfx-sound-design
description: >-
  Design retro / 8-bit game sound effects with MJSFX through its MCP tools — coins, lasers,
  explosions, jumps, power-ups, hits, UI blips, and cohesive sound families. Use this whenever the
  user wants to create, generate, or design game / arcade / retro SFX through MJSFX (e.g. "make a coin
  pickup", "I need a laser and an explosion for my game", "design a sound palette for my platformer",
  "give me a power-up sound"), or whenever the MJSFX MCP tools are available and the task is sound
  design — even if the user doesn't say "MJSFX". It covers the generate → audition → tweak → layer →
  effect → save loop, the parameter recipes that give each sound its character, and multi-voice
  layering for composite sounds. It also pushes well beyond stock presets to **invent original,
  experimental sounds and textures** — reach for it when the user wants something new, weird, unique,
  "never heard before", or just "surprise me", not only a stock effect.
---

# Designing game sound effects with MJSFX

MJSFX exposes its synthesis engine over MCP. The tools are dumb primitives — **you are the sound
designer.** Work in two modes: **reproduce** the known game-SFX vocabulary on request (coin, laser,
explosion…), and — the real payoff of a tireless AI partner — **invent** sounds that don't have names
yet by exploring the parameter space far past the presets. The recipes below are the floor; the
"Inventing new sounds" section is the ceiling. Do both, and lean into invention whenever the user is
even a little open to it.

## Prerequisite

This needs the **MJSFX MCP tools** (e.g. `generate_from_generator`, `set_params`, `render_wav`). If
they aren't available, tell the user to open MJSFX (Direct-download build) → **Settings ▸ Connect to
AI** and paste the config into their client, then retry. Everything below assumes the tools are live.

## Discover before you guess (this matters)

The tools validate against fixed lists. **Read the resources first** instead of guessing names:
- `list_generators` — the real generator IDs (they are **kebab-case**: `pickup-coin`, `laser-shoot`,
  `explosion`, `powerup`, `hit-hurt`, `jump`, `blip-select`, `randomize`, `tone`).
- `param_reference` — every `SfxParams` field, its range, and meaning, plus the waveType map.
- `list_effects` — the effects and their parameters.

If a generator/effect name is rejected, the error lists the valid options — use them, don't keep
guessing. (A wrong guess like `pickup_coin` should immediately tell you the real `pickup-coin`.)

## The contract: sounds are JSON you carry

Tools are **stateless**. Each one takes a sound (the `SavedSound`/`Composition` JSON) and returns the
updated sound; **you hold the sound and thread it through the calls.** Don't hand-author the JSON from
scratch — start from `blank_sound` or a generator (they come with valid `id` fields) and transform
with the tools. Hand-written `id`s must be real UUIDs, which is friction you avoid by using the tools.

## The loop

1. **Start** — `generate_from_generator(name, seed)` for a known family, `random_sound(seed)` to
   explore, or `blank_sound(waveType)` to author from scratch. Seeds are deterministic (same seed →
   same sound), so you can reproduce and iterate.
2. **Shape** — `set_params(sound, {...})` to set specific params; `mutate(sound, amount, seed)` to nudge
   for variations (lock params you like). For a multi-layer sound, pass `set_params(..., layer=N)` to
   edit a specific voice (default is layer 0).
3. **Layer** (optional, the differentiator) — `compose([soundA, soundB], offsets, gains)` to stack
   voices into one composite sound.
4. **Effect** (optional) — `add_effect(sound, target, effect, params)`; `target` is `"main"` or a
   layer index.
5. **Audition** — `render_wav(sound, name)` writes a WAV and returns duration + peak. Judge by those
   and by ear; `describe(sound)` gives a readable summary.
6. **Save** — `save(sound, name)` writes the JSON into the user's MJSFX library (it appears in the app).

A typical reply: generate → set a few params → render_wav so the user can hear it → save. Render early
so there's something to listen to; don't just describe.

## Character recipes — what makes each sound read as itself

These are **launchpads, not destinations** — the canonical starting point for each common sound and the
param that owns its identity. Reach for them when the user wants "a coin"; then offer to push past them
(see "Inventing new sounds"). Start from the named generator, then push the **one or two params that own
the character.** The "why" matters more than exact numbers — explain it as you go.

- **Coin / pickup** — `pickup-coin`. The signature is a **two-step upward pitch jump** partway through:
  set `arpMod` ≈ 0.4–0.6 (how far it jumps) and `arpSpeed` ≈ 0.5–0.6 (when). Square wave (`waveType`
  0) for the arcade timbre; `envPunch` ≈ 0.4 for the percussive attack; short (`envSustain` ≈ 0.05,
  `envDecay` ≈ 0.3). **Don't trust random seeds for coins** — `pickup-coin`'s arp is seed-random, so
  most rolls come out as flat blips. Set `arpMod`/`arpSpeed` explicitly; the arp *is* the coin. Higher
  `arpMod` = brighter, more triumphant ("coin get").
- **Laser / shoot** — `laser-shoot`. A fast **downward pitch sweep**: high `baseFreq`, strong negative
  `freqRamp`. Square or saw (`waveType` 0/1) for bite; short `envDecay`. A touch of `freqDramp`
  steepens the dive.
- **Explosion** — `explosion`. **Noise** (`waveType` 3), low `baseFreq`, **long `envDecay`**, downward
  `freqRamp`, and a lowpass that closes (`lpfFreq` down + `lpfRamp` negative) for the "boom → rumble."
- **Jump** — `jump`. **Upward** `freqRamp` (positive), square wave, medium-short decay. Ascending =
  "up/launch."
- **Hit / hurt** — `hit-hurt`. Short **noisy burst**, fast `envDecay`, little/no sustain — an impact,
  not a tone.
- **Power-up** — `powerup`. A **rising arpeggio** (`arpMod` up, repeated via `repeatSpeed`) or a long
  upward `freqRamp`; longer than a coin; often richer when layered (below).
- **UI blip / select** — `blip-select`, or a `tone` with **no arp**: very short (`envDecay` ≈ 0.05–0.1),
  single pitch. Removing the arp is the point — it reads as a tick, not a reward.

Direction of pitch is near-universal grammar: **up = positive/reward, down = damage/loss.**

## Layering — composite sounds (the multi-voice edge)

Single oscillators only go so far; real game sounds are often several voices. `compose` assembles them:

- **Stacked (simultaneous, offset 0)** — richer timbre: e.g. a square + a triangle an octave down for
  body, or noise under a tone for grit.
- **Staggered (`startOffsetSec`)** — sequences/arpeggios/melodies: a 3-note coin jingle = three short
  tones at rising `baseFreq`, offset ~0.06–0.1s apart.
- **Per-layer `gain`** — balance so one voice doesn't bury the others; `masterGain` trims the whole.

This is exactly how MJSFX recreates layered game sounds — lean on it for anything beyond a single blip.

## Effects

`add_effect(sound, target, effect, params)` — effects: `bitcrush` (grittier/lower-fi, that console
texture), `distortion`, `tremolo`, `phaser`, `delay`, `reverb` (space/chime tail), `adsr`. `target`
is `"main"` (whole sound) or a **layer index** (one voice). Order = signal flow. `set_effect` /
`remove_effect` edit the chain. Use the **exact param names from `list_effects`** (e.g. reverb is
`mix`/`size`/`damping`/`decay`, not `wet`/`roomSize`) — wrong names now error with the valid list.
Effects are active when added; pass `enabled: 0` to add one bypassed. Integer-typed params
(`bitcrush.downsample`, `phaser.stages`) expect whole numbers.

## Inventing new sounds — the real point

Reproducing the canon is table stakes. The reason to have a tireless AI sound designer is to make
sounds that **don't have names yet** — and you can audition 30 variants in the time a human tries
three. When the user wants something "new", "original", "weird", "unique", or "surprise me" — and
even when they just asked for a stock sound — shift from recipe-following to exploration and offer the
off-recipe option. Be bold and generative; a safe preset is the least interesting thing you can return.

Techniques that actually find new sounds:

- **Evolve with locks — for fine siblings, not big leaps.** `mutate(sound, seed, amount, lock=[the
  params you like])` applies *small* per-param jitters (faithful to sfxr — even `amount` 2 moves each
  param only a few hundredths, and it never changes `waveType`). So `mutate` makes a *family of close
  siblings*, not new sounds. **Pass a different `seed` each call (1, 2, 3 …)** — same sound + same seed
  is deterministic. Lock the keeper's good params, bump the seed, mutate again to refine. For genuinely
  BIG variety or new timbres, do NOT lean on `mutate` — use the next two levers (roll `random_sound`
  wide, or `set_params` to deliberate extremes / a new `waveType`).
- **Roll wide, chase the outlier.** Generate 10–20 with `random_sound` (varied seeds) or repeated
  `mutate`, render them all, and follow the one that's *surprising* — not the one nearest a cliché.
  Breadth-then-curation is your superpower; the human can't explore that many by hand.
- **Go past the preset ranges.** Recipes use safe values; novelty lives at the extremes and in
  combinations presets avoid — heavy `vibStrength`/`vibSpeed`, fast `dutyRamp`, aggressive
  `repeatSpeed` retriggers, extreme `arpMod`, steep `freqDramp`, hard filter sweeps. Borrow a param
  out of context (an explosion's noise under a coin's arp) and listen.
- **Use the whole palette.** MJSFX adds waveforms beyond sfxr's core — triangle plus several colored /
  bit noises (`waveType` 4–8; confirm the live map in `param_reference`). The canon ignores them; they
  are fertile ground for timbres no preset makes.
- **Layer unrelated voices.** `compose` isn't only for realistic composites — stack a tonal sweep +
  colored noise + a detuned octave for textures, drones, glitches, or sounds that *morph* across their
  length via staggered `startOffsetSec`. Single oscillators can't do this; layering is where the truly
  novel stuff lives.
- **Transform with effects.** `bitcrush`, `phaser`, `distortion`, `delay`, `reverb` turn a plain tone
  alien. Chain them, apply per-layer, push them harder than "tasteful" — FX is where a boring source
  becomes a signature.
- **Design from a feeling, not a category.** "ominous", "juicy", "glassy", "broken machine",
  "underwater" — translate the adjective into params (inharmonic layers + slow attack = ominous; short
  + punchy + bright = juicy; high sine + long reverb = glassy) instead of grabbing the nearest preset.
- **Chase accidents.** Render constantly and *listen*. When something is wrong-but-interesting, steer
  *into* it — the best original sounds are usually happy accidents you refine, not things you specced.

Then curate: render the contenders, keep a few with genuinely distinct character, name *why* each is
interesting, and let the user pick or ask for another pass. A handful of bold, varied options beats one
safe preset every time — and "I tried 25 and these three are the keepers" is exactly the work a human
can't do alone.

## Building a cohesive family

When asked for several sounds for one game ("coin, gem, power-up, hurt"), keep them **in the same
timbre family** — same `waveType`, related envelopes, consistent loudness — so they feel like one
game. Vary pitch/arp/decay for distinct events, not the fundamental character. Offer the user a couple
of graded variants (e.g. brighter vs richer) and say which fits which use (routine pickup vs milestone)
— and watch duration: a long chime smears if the player triggers it rapidly.

## Good defaults for replies

- Render a WAV so the user can actually hear it; report the path + duration.
- Save the JSON so it lands in their MJSFX library for further tweaking.
- Explain the *why* behind the params you moved (briefly) — it teaches and it's checkable.
- Offer next steps (variants, a family, a different timbre) rather than stopping dead.
