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
- `list_effects` — each effect's params with their **type, range, unit, and default** (e.g. delay
  `timeMs` is in ms over 1–1000, phaser `stages` is an int 2–12). Read it before setting effect params.
- `exemplars` — an annotated **real composition** showing the architecture that makes invented
  sounds read as alive (per-layer FX chains, LFOs on FX params, staggered entrances). Read it
  before designing anything beyond a stock preset — one exemplar teaches what pages of prose don't.

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
6. **Save** — `save(sound, name)` writes a `.mjsfx` file into the user's MJSFX library (it appears in the app).
   From app v1.15 the save contract changed: saving an existing name **dedupes by default** ("coin" →
   "coin 2") instead of overwriting; pass `overwrite: true` to update a sound in place. So the iterate
   loop is: first save plain, then re-saves of the same sound use `overwrite: true` (otherwise you
   accumulate "name 2", "name 3" files). On v1.14-and-earlier servers `overwrite` doesn't exist and
   save overwrites silently — if the param is rejected, you're on the old server; just save plain.
7. **Reload** (across turns/sessions) — `list_sounds()` lists saved names; `load(name)` returns a saved
   sound so you can keep iterating on it later without re-deriving it. (From v1.15, `load` returns the
   sound's `name` synced to its filename stem — trust it as the re-save target.)

A typical reply: generate → set a few params → render_wav so the user can hear it → save. Render early
so there's something to listen to; don't just describe.

**Loudness — balance a family with `normalize`.** When you make several sounds meant to sit together
(a UI set, a weapon family), their raw peaks vary a lot. `normalize(sound, targetPeak)` sets the sound's
master gain so its rendered peak hits `targetPeak` (default 0.8). Normalize each to the **same**
`targetPeak` before saving so they're consistently loud. It returns the updated sound — chain it into
`save`/`render_wav` like any other transform.

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
`list_effects` also gives each param's range and unit, so set ms/Hz values (delay `timeMs`, tremolo
`rate`) in real units, not 0–1.
Effects are active when added; pass `enabled: 0` to add one bypassed. Integer-typed params
(`bitcrush.downsample`, `phaser.stages`) expect whole numbers.

Two habits that separate flat results from deep ones: **put FX on layers, not the main bus** — a
main-bus chain processes every voice identically (a blunt instrument), while per-layer chains let
each voice occupy its own sonic space; and **use a per-layer `adsr` unit for articulation** — it
re-shapes that voice's amplitude *after* synthesis, finer than the source envelope allows, and a
long `release` gives a voice its own tail.

## Modulation — LFOs (movement over time)

Static sounds read as flat; the ones that feel alive **move**. An LFO drives a parameter up and down
over the sound's life — vibrato, tremolo, filter wub, PWM, a pitch dive, a shimmering tail. Four tools:
`add_lfo` / `set_lfo` / `remove_lfo` / `list_lfos`, attached to a **layer index** (`"0"`, `"1"`, …) or
the whole sound (`"main"`). Every LFO gets a UUID — read it from `list_lfos` to edit or route it.

What to reach for (the `target` address routes the LFO's output within that layer):
- **`source.baseFreq`** — vibrato (slow sine, small depth) or a pitch dive/rise (one-shot, big depth).
- **`source.duty`** on a square — PWM, that hollow sweeping buzz.
- **`source.lpfFreq`** — filter wobble ("wub") or an opening sweep.
- **`gain`** — tremolo (sine) or a stutter/gate (fast square).
- **`fx.<unitId>.<key>`** — modulate an effect itself: wobble a phaser, swell a reverb `mix`, drift a
  delay `timeMs`. Read `<unitId>` from the sound JSON (each FX unit's `id` field in `fx.units`).

Fields that matter for craft (full table in the MCP guide's **"LFO modulation"**):
- **`rateMode`** — `hz` for an absolute speed; **`cycles`** ties the rate to the sound's length, so a
  sweep lands exactly once (or N times) regardless of duration.
- **`cycleMode: "oneShot"`** runs the shape once then holds — an envelope-synced sweep or a single
  pitch drop — vs `loop` for ongoing wobble/tremolo.
- **`shape`** — `sine` (smooth), `square` (gate/stutter), `sampleHold` (random steps, bleepy arps),
  `sawUp`/`sawDown` (ramps).
- **`depth`** is signed −2…2; ±1 is already a full swing for gain/FX/master (more over-drives them),
  bigger depths buy extra pitch range. Set **`enabled: 1`** to activate (LFOs add disabled).

**LFOs can modulate each other** — route one to another's `rate`/`depth`/`phase` (`mod.<modId>.rate`,
`<modId>` from `list_lfos`). A slow LFO sweeping a fast one's rate gives evolving, never-quite-repeating
motion — the difference between a sound and a *texture*. Reach for it on anything meant to feel organic
or alien. LFOs persist in the saved sound (part of the design, not a render-time setting).

When a request implies motion — "wobbly", "warbly", "throbbing", "shimmering", "alive", "evolving",
"siren", "laser sweep" — it's an LFO, not just a static patch.

## Inventing new sounds — the real point

Reproducing the canon is table stakes. The reason to have a tireless AI sound designer is to make
sounds that **don't have names yet** — and you can audition 30 variants in the time a human tries
three. When the user wants something "new", "original", "weird", "unique", or "surprise me" — and
even when they just asked for a stock sound — shift from recipe-following to exploration and offer the
off-recipe option. Be bold and generative; a safe preset is the least interesting thing you can return.

**First: read the `exemplars` resource.** It is a real composition that already solves the "how do I
make something sound intentional, not preset-shaped" problem — four voices, each with its own FX
chain, LFOs animating the *effects* (a bitcrush's `jitter` and `bits`, a filter sweep), per-layer
`adsr` articulation, staggered entrances. Movement in the **processing** is what reads as alive;
static FX on a single source is what reads as basic. Steal its architecture, not its params.

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
- **Modulate for motion.** Static params freeze a sound in one shape; a moving param makes it evolve
  across its own duration — vibrato, a PWM sweep, filter wub, a pitch dive that lands exactly at the
  end (`cycleMode: "oneShot"` + `rateMode: "cycles"` ties the sweep to the sound's length so it feels
  composed, not random). The real invention lever is **LFO→LFO**: route a slow modulator to another's
  `rate` or `depth` and the two drift against each other, generating motion that never quite repeats —
  closer to a living texture than a sample. `sampleHold` gives random stepped pitch jumps for glitch
  and disorder. Push these past tasteful: a deep pitch LFO on a noise layer reads as something
  breathing; a fast filter LFO on a square reads like a machine malfunctioning.
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
- Save the sound (a `.mjsfx` file) so it lands in their MJSFX library for further tweaking.
- Explain the *why* behind the params you moved (briefly) — it teaches and it's checkable.
- Offer next steps (variants, a family, a different timbre) rather than stopping dead.
