# Critteria — Spec

A Tamagotchi-style virtual pet app for four siblings, built as a web app/PWA
(Silk browser on Fire tablets), served from a Cloudflare Worker at
`critteria.immotus.app`. Formerly the `pet/` directory of
`NoliCommoveri/site`; extracted to its own repo on 2026-07-30 via
`git subtree split -P pet`, so pre-split history lives on `main`.

## 1. Current prototype

`index.html` is a working local-only prototype:
- Stats (hunger, happiness, energy, cleanliness) decay based on real elapsed
  time (computed from a stored timestamp), so the pet keeps "aging" even
  while the tablet is closed.
- Actions: Feed, Play, Clean, Sleep.
- Species picker: real AI-generated pixel-art stills for T-Rex, unicorn,
  dragon, hippocampus, pegasus, and wolf (`assets/`) — see Art pipeline
  below. Species is chosen once at first launch and then locked for that
  pet — see Species selection below. This is the full kept roster (all six
  conform to the Art pipeline rules below); phoenix, griffin, and
  slime/blob were legacy placeholders and have been removed from the
  picker, `SPECIES_POSES`, and (for phoenix/griffin) the asset files
  themselves.
- Mood- and action-driven **pose swapping**: the renderer picks a distinct
  drawn sprite per state (idle/happy/sad/sleep + eat/eat-chew/play/bath) from
  the `SPECIES_POSES` manifest, falling back to `idle` for any pose not yet
  drawn — see Art pipeline below. Every kept species has all four required
  poses (idle/happy/sad/sleep); action poses (eat/eat-chew/play/bath) are
  still unbuilt for all of them, so the fallback chain does that work.
- `localStorage` persistence.

Everything below is the target end-state, to be built in phases on top of
this prototype.

## 2. Architecture

```
[Fire tablet Silk browser / PWA]
        |  fetch (poll every ~5s while foregrounded)
        v
[Cloudflare Worker on critteria.immotus.app]
        |-- [assets] binding → serves index.html, assets/, sw.js
        |-- functions/api/*  → API routes bundled into dist/worker/index.js
        |-- [[d1_databases]] → D1 (SQLite): families, kids, devices,
                                pairing_codes, pets, pet_events,
                                helper_action_usage, plus deferred
                                catalog tables (§3)
```

One Worker serves the PWA and the API from the same origin — no CORS, no
separate `api.*` subdomain. Deploys are git-connected to `main`; every push
rebuilds and redeploys automatically. Deployment mechanics and every
Cloudflare gotcha we've hit are in §9.

Adapted from `NoliCommoveri/Star-homeschool`, which is running the same
architecture — see its `docs/parent-sync-spec.md` for the reasoning-in-full.
The chosen shape is deliberately smaller than an earlier draft of this spec,
which called for Cloudflare Pages + Durable Objects + WebSocket fanout + KV
+ Web Push. That draft is superseded — see the **Deferred** note below for
what it becomes instead.

**Deferred to a later phase**: Durable Objects (per-pet authoritative live
state) and WebSocket fanout are the target end-state for real-time co-play,
but polling reads with last-write-wins on D1 are more than enough for a
family taking turns caring for each other's pets. If polling latency ever
bothers anyone, upgrading to DO + WebSocket is a transport change, not a
data-model change. Also deferred: KV (not needed — tokens live in the
`devices` table) and Web Push (nice-to-have; §8 Phase 5).

Cloudflare **Pages** is not part of this — it was folded into Workers
entirely, and "Pages project" in older writing (including earlier drafts of
this file) now just means "Worker with `[assets]` binding."

## 3. Data model (D1)

Only one family exists per deployment — creating a family requires the
`SIGNUP_SECRET` env var (§7, §9 Step 7). Additional kid devices are paired
via short-lived 6-char codes (single-use). Auth model is adapted from
`NoliCommoveri/Star-homeschool` `docs/parent-sync-spec.md §6`.

Live tables (v1):

- `families` — id (128-bit random hex, never derived), created_at.
- `kids` — id, family_id, display_name, avatar (preset key or emoji),
  pin_hash (SHA-256 of 4-digit PIN + kid_id salt; nullable in v1),
  created_at.
- `devices` — id (client-generated UUID, stable across re-pairing),
  family_id, kid_id (null until claimed by a kid at first-run),
  token_hash (SHA-256; raw token never stored; token rotates while id
  does not), label ("Ana's tablet"), created_at, last_seen, revoked.
- `pairing_codes` — code_hash (SHA-256 of 6-char code), family_id,
  expires_at (5-min TTL), used_at (non-null once redeemed; single use).
- `pets` — id, kid_id (one pet per kid, UNIQUE), species, color_variant,
  name, stage, created_at, hunger, happiness, energy, cleanliness,
  sleeping, last_updated (server epoch ms; **decay is computed lazily on
  every read/write**, same pattern as the current `localStorage`
  prototype's `applyDecay` — see `index.html`).
- `pet_events` — id (autoincrement), pet_id, actor_kid_id, action
  (`feed`/`play`/`clean`/`sleep`/`wake`/`visit`/`gift`), occurred_at,
  detail (optional JSON — gift contents etc.). Append-only; doubles as
  audit trail, visit history, and presence signal (any event by a
  kid within the last ~30 s = "currently visiting" — `actor_kid_id` is
  what's stored on each event, not device_id). `visits` from
  an earlier draft is dropped — presence and visit history come out of
  `pet_events` alone, saving a second write per session.
- `helper_action_usage` — visitor_kid_id, host_pet_id, action, utc_day
  ('YYYY-MM-DD'), used_at. **Primary key across all four** so a single
  `INSERT OR ABORT` either succeeds or fails on the daily rate limit
  with no race window — do not use `INSERT OR IGNORE`, which would
  silently succeed on the duplicate and let the helper action run.
  Day boundary is UTC deliberately (one column, no per-family config),
  so on the US west coast the daily reset lands mid-afternoon local —
  fine for one family with a stable timezone; if this ever feels wrong,
  the fix is a `families.tz` column and computing the day string from
  it, not schema changes to this table.

Deferred tables (Phase 5, §8) — four of them, kept in the schema so a
future migration doesn't need to reshape any of the above:

- `pet_equipped_items` — pet_id, slot, item_id.
- `items` (catalog) — id, slot (head/neck/back/…), name, image_asset,
  species_compat (JSON array), unlock_type (points/streak/gift).
- `locations` (catalog) — id, name, image_asset, unlock_type.
- `pet_current_location` — pet_id, location_id.

The concrete `CREATE TABLE` statements are inlined in §9 Step 5 so they
can be dropped straight into `schema.sql`.

## 4. Art pipeline

Because hand-coded SVG shapes don't read as "real" to young kids, the art is
pixel-art stills with transparent backgrounds, generated (ChatGPT image gen)
and then cleaned up to the constraints below.

**Mood is conveyed by a different drawn pose, not by filtering one image.**
An earlier version of this spec said "a single idle/happy pose per species is
enough" with other moods faked via CSS. That was wrong in a way kids notice
immediately: a desaturated, slightly rotated idle sprite with a floating "Z"
next to it is still a pet standing up with its eyes wide open. CSS can change
color, timing, and position — it cannot close an eyelid, droop an ear, or lie
a creature down. Distinct pose art is the deliverable.

### Style constraints (normative)

Every sprite, for every species and pose, must satisfy:

- **128×128 canvas**, creature filling roughly 75–85% of it, leaving room for
  tails/wings/ears/effects. Never crop body parts.

  Relaxed from the style bible's 64×64 deliberately. Two measurements drove
  it: the app renders the pet at `max-width: 220px`, and generated output
  lands at ~150px of native detail per creature (measured: the dominant
  pixel-block run length in the first T-Rex sheet was 2px across a 384px
  cell, so a ~192×256 native grid). Forcing that down to 64 destroys the
  shading and small features — teeth, claws, eye highlights — and means
  redrawing rather than cleaning up. 128 is still crisp at 220px display, is
  close to what the generator natively produces, and cuts the cleanup cost
  per asset substantially. The cost is that it reads less chunky-retro than
  a true 16-bit 64×64 sprite would.
- **One shared palette** across all sprites, environments, UI, and effects.
  Committed as a file, not held in someone's head.
- **Three-quarter view, ~20–30° rotation**, facing slightly toward the
  viewer. Never full profile, never top-down.
- **Light source upper-left**, consistently. 2–3 shadow values plus 1
  highlight value. No gradients.
- **Dark outline, never pure black**, hue-shifted toward the creature's own
  palette.
- **No anti-aliasing, no blurry shading, no AI-painted texture.** Generated
  output gets downsampled, quantized to the palette, and hand-cleaned; it is
  a starting point, not the asset.
- **Pose registration:** all poses of one species share canvas size, ground
  line, and body scale, so swapping poses doesn't make the pet visibly jump
  or resize. This is the constraint most likely to be violated by generating
  each pose as an independent image, and the most jarring when it is.

`trex.png` and `trex-happy.png` are the first conforming assets: 128×128,
20-colour shared palette, identical scale and ground line (baseline row 117,
creature 104px tall = 81% of canvas), hue-shifted outline. They came from
cells 1–2 of a generated 8-pose sheet, via the cleanup below. `trex-sad.png`
and `trex-sleep.png` followed later, generated on their own attaching the
by-then-growing reference and quantized onto the locked ramp; T-Rex now has
all four required poses. `reference/trex-palette.txt`/`.png` hold the ramp
and `reference/trex-reference-4x.png` holds all four poses (4x) for
attaching when T-Rex's action poses are generated next.

`dragon.png` (idle) is the second conforming asset and the first *new*
species built to these rules from the start: 128×128, purple/blue 19-colour
palette (`reference/dragon-palette.txt`/`.png`), same baseline row 117 and
81% canvas fill as the T-Rex registration, hue-shifted outline. It was
generated as a single pose (no sheet, no reference image to attach — nothing
existed yet for this species), so the cleanup pipeline established the crop
window and palette directly rather than inheriting one.

`dragon-happy.png` followed, generated on its own (not sheeted with idle)
and attaching `dragon-reference-4x.png` for style/proportions only, so its
scale had to be independently derived and verified per step 4 below rather
than inherited from a shared crop window — it landed on the exact same
top/bottom rows as idle (13–116, 104px tall) with no adjustment needed, and
was quantized onto the *locked* `dragon-palette.txt` rather than a new ramp.

`dragon-sad.png` followed the same process — generated on its own attaching
the (by then two-pose) reference, independently scaled and ghost-overlay
verified against idle (again landed on rows 13–116 with no adjustment),
quantized onto the locked palette.

`dragon-sleep.png` completed the required tier. Curled poses need the
different scaling approach flagged above: matching canvas *height* would
undersize it (a curled sleeper is short and wide), so it was scaled by
*width* instead, landing at 81% width fill — matching what `trex-sleep.png`
independently landed on (81.2%) — with the same bottom row (116) as the
other three dragon poses, ground line held. Ghost-overlay verified the head
reads the same scale as idle before quantizing onto the locked palette.

`dragon` now has all four required poses. `reference/dragon-reference-4x.png`
holds all four (idle/happy/sad/sleep, 4x) and is what gets attached when
generating `dragon`'s action poses next.

`unicorn` has also been completed to these rules (idle/happy/sad/sleep, same
128×128/81%-fill registration), it just predates this paragraph being
written up. `reference/unicorn-palette.txt`/`.png` hold the locked pink-on-
pink ramp and `reference/unicorn-reference-4x.png` holds all four poses
(4x) for attaching when unicorn's action poses are generated next.

`hippocampus.png` (idle) is a conforming asset built the same way as dragon's
first pose: generated as a single pose (no sheet, no conforming reference to
attach yet), so crop window and palette were derived fresh — rows 13–116
(104px tall, 81% canvas fill), matching the T-Rex/dragon baseline
registration. Small fins were added on the back of each front leg, above the
hoof, to read as seahorse anatomy consistent with the tail and mane fins.
The raw generation came back noticeably softer/gradient-shaded than T-Rex
and dragon's source art — two rounds of prompting to push flatter
cel-shading closed most of that gap, and the remainder was closed in
cleanup: quantized to a 10-colour teal/aqua ramp
(`reference/hippocampus-palette.txt`/`.png`), with the near-black
quantization cluster collapsed to one hue-shifted dark-teal outline rather
than left as several near-duplicate near-blacks. Fewer colours than T-Rex/
dragon's ~19–20 because the creature only has two body hues (white, teal)
against their three-plus; the flat-banding rule, not a specific count, is
what's normative.

`hippocampus-happy.png` followed, generated on its own attaching the idle
reference and asking for the pose only (bright eyes, open mouth, lifted
posture) with everything else — scale, ground line, palette, flat-shading
technique — held fixed. It needed its own crop window rather than reusing
idle's verbatim, since the lifted front leg extends beyond idle's silhouette
on the left and top, but landed within a couple of pixels of idle's window
anyway (same target row 13–116) — confirmed by bounding-box comparison and
a ghost-overlay check before cleanup, not just assumed. Quantized onto the
locked `hippocampus-palette.txt` rather than a new ramp; all 10 colours got
used, nothing fell off-palette.

`hippocampus-sad.png` followed the same process: generated on its own
attaching the two-pose reference, asked to name the stance explicitly
(standing, weight back, head/ears lowered, tail coiled low, eyes
half-closed) rather than left to default to lying down. Registration held —
landed one pixel off idle/happy's horizontal placement (x=30 vs 31, both
top row 13) — but cleanup hit a real bug the first pass: the crop was taken
from the pre-key RGB image instead of the post-key RGBA one, so the magenta
margin inside the crop box came through fully opaque and quantized into a
visible teal rectangle behind the creature. Re-run correctly (key to alpha
*before* cropping, exactly as idle/happy did), it quantized cleanly onto
all 10 locked palette colours with no rectangle. Worth watching for on
future poses since it's an easy step to skip by accident.

`hippocampus-sleep.png` completed the required tier. Curled poses need the
different scaling approach flagged above: scaled by *width* rather than
height, landing at 80.5% width fill (bbox 103px wide within the 128
canvas) — matching what `trex-sleep.png` and `dragon-sleep.png` both
independently landed on (80.5% width fill exactly) — with the same bottom
row (116) as the other three hippocampus poses, ground line held.
Ghost-overlay verified the head reads the same scale as idle before
quantizing onto the locked palette; all 10 colours got used again, no
repeat of the sad-pose cropping bug.

`hippocampus` now has all four required poses.
`reference/hippocampus-reference-4x.png` holds all four (idle/happy/sad/
sleep, 4x, 2048×512) and is what gets attached when generating
`hippocampus`'s action poses (`eat`/`eat-chew`/`play`/`bath`) next.

`pegasus.png` (idle) replaces the old placeholder outright rather than being
re-cut from it — the legacy still was a ~600px anti-aliased white-and-gold
side-ish illustration with a plain dark outline, none of which conforms.
The new one is a from-scratch conforming asset, built the same way `dragon`
was: generated as a single pose (no sheet, no prior reference) on a flat
magenta background, then cleaned up (magenta-tolerant flood fill from the
border, box-filter downsample, crop to the shared 128×128/row-14–117
registration, median-cut to 20 colours). Two raw-quantization artifacts
needed a manual fix before committing: the anti-aliased boundary ring had
picked up a magenta-shifted maroon tint instead of a true outline colour,
and the pupil quantized to pure black — both recoloured to hue-shifted dark
browns per the cleanup pass's step 6, consistent with how every other
species avoids literal `#000000`. Colour family is palomino: golden-tan
coat against a lighter flaxen-cream mane/tail/wings/hooves, the same
two-tone-within-one-hue-family pattern as the pink-on-pink unicorn and the
purple-on-blue dragon. `reference/pegasus-palette.txt`/`.png` hold the
locked 20-colour ramp and `reference/pegasus-reference-4x.png` is what gets
attached when generating `pegasus`'s `happy`/`sad`/`sleep` poses next.

`pegasus-happy.png` followed: raised/spread wings, a lifted front hoof,
and an open-mouth smile, generated on its own attaching the idle reference
and independently scaled per step 4 (its raw silhouette ran wider than
idle's, from the spread wings and raised leg, but the topmost point was
still the same forelock cowlick as idle in both, confirming the height
anchor held) — landed on the exact same top/bottom rows as idle (14–117,
104px tall) with no adjustment needed. Ghost-overlay verified the head
reads the same scale as idle before quantizing onto the *locked*
`pegasus-palette.txt` rather than deriving a new one; every opaque pixel
mapped onto an existing palette entry with no new colours and no stray
near-black. `reference/pegasus-reference-4x.png` now holds both poses
(idle/happy, 4x) and is what gets attached when generating `sad` next.

`pegasus-sad.png` needed a real fix mid-pipeline, not just a scale
judgment call: the border flood fill (adaptive per-step colour-distance,
the method that had worked cleanly for idle and happy) tunnelled through
a smooth highlight gradient on the body and ate most of the interior,
leaving only outline strokes — visible immediately as a speckled,
mostly-transparent result once quantized. Root cause was the same one the
cleanup pass's step 2 already warns about for outline gaps: an adaptive
neighbour-distance rule has no fixed floor, so a long enough chain of
small steps crosses the whole body. Fixed by reclassifying background with
a fixed rule (green channel < 55, since magenta's is ~0–4 and every body
and outline colour here is well above that) inside the same border-flood
fill, which keeps the connectivity property (isolated dark interior
pixels — the pupil, eyebrow lines — can't be reached from the border) while
removing the gradient-tunnelling failure mode.

Scale was the second issue: this pose's raw silhouette (head drooped, wings
folded low) is genuinely shorter than idle's, not just differently cropped,
so matching idle's exact 104px height would have inflated the whole body —
confirmed by overlaying idle's traced outline on candidates at a few
heights, where 104px sat visibly outside the sad body's legs and 97px
(76% canvas fill, still inside the 75–85% band) tracked them closely.
Landed on the same bottom row (117) as idle and happy, so the ground line
still holds. `reference/pegasus-reference-4x.png` now holds all three
poses (idle/happy/sad, 4x) and is what gets attached when generating
`sleep` next, the last pose in the required tier.

`pegasus-sleep.png` completed the required tier. The same fixed-threshold
border flood fill used for `sad` carried over cleanly here (no gradient
leak this time either). Curled poses scale by *width*, not height, per the
same reasoning as `trex-sleep`/`dragon-sleep`: this pose's raw silhouette
already landed at 80.3% width fill without any correction needed, scaled
to 104px wide (81.25%, matching `trex-sleep`'s 81.2% almost exactly) and
placed on the same bottom row (117) as the other three poses. `pegasus`
now has all four required poses. `reference/pegasus-reference-4x.png`
holds all four (idle/happy/sad/sleep, 4x) and is what gets attached when
generating `pegasus`'s action poses next.

`hippocampus` and `pegasus` were built out to these rules independently, in
parallel. Both are keepers.

`wolf` is the sixth keeper, also built to these rules — all four required
poses (idle/happy/sad/sleep, 128×128, shared ground line, hue-shifted
outline) exist on disk, with `reference/wolf-palette.txt`/`.png` holding
the locked ramp and `reference/wolf-reference-4x.png` holding all four
poses (4x) for attaching when its action poses are generated next. Wired
into `SPECIES_POSES` and the picker.

The roster is therefore **T-Rex, unicorn, dragon, hippocampus, pegasus,
and wolf**, all six at the required four-pose tier. `griffin`, `phoenix`,
and `slime/blob` were dropped: griffin and phoenix predated these rules —
471–640px, anti-aliased, gradient-shaded, and inconsistent in camera angle
(the griffin painterly ¾, formerly the unicorn full side) — and weren't
re-cut; blob was the placeholder SVG that predated the pixel-art pipeline
entirely. That removal (species picker, `SPECIES_POSES`, and the asset
files themselves for phoenix/griffin; the inline SVG markup, CSS, and JS
branch for blob) is done.

### Cleanup pass for generated art

What a raw generated sheet needs before it is an asset, in order — each step
exists because the first T-Rex sheet failed it:

1. **Threshold alpha** to 0/255, or **key the magenta** when the generation
   used a flat `#FF00FF` background (background = red and blue both high,
   green low). Magenta is much the easier case: `trex-sleep` came back with
   1.13M clean background pixels and only ~3k contaminated fringe, against the
   transparent-background sheet's haze of ~250k partial-alpha pixels. Step 5's
   quantization de-fringes the leftovers for free — magenta has nowhere to
   snap but a real palette colour.
2. **Flood-fill from the border, treating the dark outline as a wall**, to
   separate creature from background — only needed for transparent-background
   output; a magenta key already gives a clean silhouette. **Dilate the
   outline by 1px first** —
   undilated, a hairline gap in one cell's outline let the fill into the body
   and ate 40k pixels of it.
3. **Remove baked ground/shadow.** The generator draws a tan ground patch
   that *overlaps* the toes, so cropping by row would cut feet off. In that
   band the legs are green (`g` is the max channel) and the ground is warm
   tan (`r` max, `r > b+35`, luminance 70–225) — that separates them.
4. **One shared crop window across all poses of a species**, not each
   sprite's own bounding box. This is what actually enforces pose
   registration: identical scale, identical ground line, and the generator's
   own relative placement inside the cell preserved.

   This only works for poses generated *together*. A pose generated on its
   own has no shared window, so its scale has to be derived — and matching
   the canvas height is wrong, because a curled sleeping creature is short
   and wide where a standing one is tall and narrow. Silhouette area is also
   a poor anchor: a curled pose is a denser blob (78% bbox fill vs 64%
   standing), so area-matching oversizes it. What worked for `trex-sleep`:
   scale so the sprite lands inside the 75–85% canvas-fill rule, then verify
   by rendering it with the `idle` sprite ghosted behind at 28% opacity and
   checking that the head reads the same size and the ground line holds.
   Prefer generating poses in pairs so the shared window does this for you.
5. **Downsample by box filter with ≥50% coverage** keeping a pixel, so edges
   stay hard, then quantize. For a species' *first* poses, median-cut to ~20
   colours built from all of them at once. For every pose after that,
   **quantize to the colours already committed for that species** rather than
   deriving a new palette — that is what stops poses drifting apart, and it
   doubles as the de-fringe step.
6. **Recolour near-black palette entries** to the hue-shifted outline colour.

Verify with `?pose=<name>` and check that the baseline row does not move
between poses.

### Pose set

Trimmed from a full 12-mood expression grid down to the states the app
actually has. Two tiers:

| Pose | Trigger | Depicts | Tier |
|---|---|---|---|
| `idle` | neutral mood | Standing, calm, eyes open, ¾ view | **required** |
| `happy` | mood avg ≥ 70 | Bright-eyed, open mouth, lifted posture | **required** |
| `sad` | any stat < 20 | Head/ears down, drooped tail, downturned mouth | **required** |
| `sleep` | sleep toggled on | **Lying down, eyes closed** — curled, head resting | **required** |
| `eat` | Feed tapped | Head toward food, mouth open, jaw down | action |
| `eat-chew` | alternates with `eat` | Same pose, **mouth closed / jaw up** — the second chew frame | action |
| `play` | Play tapped | Mid-bounce, weight forward toward the ball | action |
| `bath` | Clean tapped | Sitting, eyes squeezed shut, head slightly turned | action |

Required poses persist for as long as the mood holds. Action poses are
transient — they hold `ACTION_POSE_MS` (1.8s) and then hand back to the mood
pose. Eight poses × six species = 48 creature sprites, plus the item sprites
(below); that is the real art budget for this feature, and it is the reason
for the tiering.

### Action choreography

An action is not a pose swap — it is a short sequence, modelled on how
Pokémon plays item use: **the item travels to the pet, then the pet reacts.**
Tapping Feed and having the pet silently switch to a chewing stance reads as
a glitch; the arriving object is what makes the tap feel like it did
something.

Three phases inside the existing 1.8s window:

| Phase | Duration | What happens |
|---|---|---|
| 1. Approach | ~400ms | Item sprite enters from the stage edge and arcs toward its anchor point on the pet — ease-out, slight overshoot on arrival. Pet still in its mood pose. |
| 2. React | ~1000ms | Item lands and disappears (or stays put, for the bath suds). Pet switches to the action pose and plays its reaction loop. |
| 3. Settle | ~400ms | Pet eases back to the mood pose, which has by then been recomputed from the improved stats. |

Reaction loops, per action:

- **Feed → chewing.** Alternate `eat` and `eat-chew` every ~180ms for the
  duration. This is the one reaction that genuinely needs a second drawn
  frame: chewing is a change of mouth and jaw shape, and no transform
  produces it. Two frames is enough — pixel art reads chewing from a 2-frame
  loop, and a third adds cost without adding legibility.
- **Clean → shaking.** A rapid low-amplitude horizontal wobble on the `bath`
  sprite (±3–4px, ~90ms period, decaying). Deliberately **CSS-only, no
  second frame**: a whole-body shake *is* a rigid-body transform, so a
  transform is the honest implementation rather than a fake. Contrast with
  chewing above — the test is whether the shape changes or merely moves.
- **Play → bounce.** Same reasoning as the shake: an existing vertical
  transform on the `play` sprite, no second frame.

Item sprites are **64×64** (half the creature canvas, scaled with it when the
canvas rule changed) but, unlike the pose art, they're **species-specific for
feed and play** — a T-Rex gnaws a drumstick, a hippocampus nibbles seaweed, a
pegasus crunches a cabbage. A single shared apple/ball read as generic once
real per-species pose art existed next to it, so feed and toy items were
redone to match: each species gets two feed items and two toy items (picked
at random per tap, for variety, all drawn from the art already produced),
named `assets/<species>-food-<name>.png` / `assets/<species>-toy-<name>.png`.
`SPECIES_ITEMS` in `index.html` holds the mapping; a species with no entries
(currently wolf, which is unused) simply gets no travelling item rather than
a broken image.

**Clean stays shared.** There's nothing species-flavored about soap and
bubbles, so every species reuses one sprite — currently `assets/bubble-md.png`
— rather than multiplying art for a wash effect that looks the same on
everyone.

Items must be **separate files**, never drawn into a pose. The first
generated sheet baked food into four of its eight cells, which breaks the
approach phase entirely — there is nothing left to fly in. Per-species items
are now generated four-to-a-sheet (feed ×2, toy ×2) on a magenta chroma-key
background, then split, despilled to transparency, trimmed, and downscaled
to 64×64 before they reach `assets/` — each item its own file, as required.

Where an item lands is per-species, since a hippocampus's mouth is nowhere
near a T-Rex's: extend the accessory anchor config with an `item` slot, so
the food arrives at the mouth and the suds land over the body. Suds may also
sit as a persistent overlay for the reaction phase rather than vanishing on
contact. **Partially built** — `index.html` now animates the item in from
the lower-left corner, oversized and fading in, shrinking down to landing
scale as it settles at a generic front-of-pet anchor (left side, mid-height,
clear of the body) rather than overlapping it, holds, then fades out. That
anchor is still one fixed spot for every species, not the per-species
mouth/body point this section describes — real anchor tuning is the
remaining work here.

Deliberately out of scope for now: Sleep gets no travelling item (nothing is
being applied to the pet — the pet changes state), and no action gets a
screen-shake, flash, or particle burst. The tone in §1 is calm and cozy;
these should read as gentle, not as combat feedback.

### Fallback chain

`SPECIES_POSES` in `index.html` lists only files that exist. Anything
missing walks `POSE_FALLBACK` until it reaches `idle`, which is the one pose
a species may not ship without:

```
eat-chew → eat → happy → idle
eat      → happy → idle
play     → happy → idle
bath     → idle
sad      → idle
sleep    → idle
```

`eat-chew` falling back to `eat` degrades gracefully on its own terms: both
frames of the chew loop resolve to the same sprite, so the mouth simply
doesn't move and nothing flickers. The travelling item and the other two
reaction loops are CSS, so they work regardless of which poses exist — the
approach phase is worth building before any pose art lands.

`sad → idle` and `sleep → idle` are listed for defensive rendering only —
both are required tier, so these paths should never trigger for a keeper
species; they exist so the renderer degrades cleanly during art work in
progress rather than throwing on a lookup miss.

When a pose resolves exactly, the stage gets `pose-<name>`. When it falls
back, it also gets `pose-fallback`, and **only then** do the old CSS fakes
(desaturate-on-sad, dim-on-sleep, droop rotation) apply — real pose art must
not be filtered on top of what the artist already drew.

### Generating poses

Do **not** ask for all eight poses in one sheet. Tried once: the generator
held the requested pose and registration for exactly two cells, then drifted
— cells 3–8 came back as five variations on "lying down near food", with no
`sad`, `sleep`, `play`, or `bath` at all, and height varying 30% with the
ground line moving 14%.

Instead, treat a cleaned `idle` as canonical and **attach it as a reference
image, requesting 1–2 poses per call**, stating explicitly that canvas, scale,
and ground line must match the reference.

Attach `assets/reference/trex-reference-4x.png`, not the 128×128 asset —
it's the same two poses at 4× nearest-neighbour (1024×512), so the pixel grid
is unambiguous to the generator and both poses together demonstrate what
"same character, same ground line" means. `reference/` holds generation aids
only; nothing in it is loaded at runtime. `trex-palette.png` and
`trex-palette.txt` are the 20 colours of the current T-Rex ramp.

Also: 

- Say **"creature only — no food, no props, no grass, no ground, no shadow"**.
- Ask for a flat **magenta `#FF00FF` background** rather than transparency.
  Transparency mostly works but leaves a haze, and for a green creature that
  haze is green — the hardest possible case to key.
- For `sad`, name the stance or it collapses into lying down: *"standing
  upright on both feet, weight back, head lowered slightly, tail drooping,
  eyes half-closed"*.
- For `eat-chew`, don't request a pose. Attach the finished `eat` frame and
  ask for *"this exact image, unchanged, with only the jaw closed"*.

### Adding a pose

1. Draw/generate it to the style constraints above; save as
   `assets/<species>-<pose>.png`. (`idle` keeps the bare
   `assets/<species>.png` name it shipped with.)
2. Add the one-line entry to `SPECIES_POSES` in `index.html`. The
   manifest is explicit on purpose — deriving paths by convention would 404
   into a broken-image icon for every pose not yet drawn.
3. Check it with `index.html?pose=<pose>`, which pins that pose and its
   mood so the art can be reviewed without starving the pet into the state.

Sprites ≤128px wide automatically get `image-rendering: pixelated` on load,
so conforming 128×128 art upscales crisply at the 220px display size while
the legacy 471–640px stills keep smooth downscaling. That detection is
transitional and can be dropped once every asset is 128×128.

### Layering

Rendering is layered, back to front:
1. Location/background image (behind everything)
2. Species pose sprite for the current state, tinted for the chosen color
   variant (see §5)
3. Accessory overlays (hat, scarf, etc.), positioned via a per-species
   anchor config — now keyed `{ species, stage, pose } → { slot: {x, y,
   scale} }`, since an accessory sits differently on a standing pet than a
   sleeping one
4. State effects on top (sparkle, Zzz, dirt smudges)

Additional sprites needed over time: the pose set per species per lifecycle
stage (egg/hatchling/young/teen/adult), plus one image per accessory item.
Egg stage needs only `idle`. Locations are just background images — no
per-species work.

## 5. End-state feature breakdown

### Lifecycles (aging)
`pets.stage` advances egg → hatchling → young → teen → adult based on
elapsed real time since `created_at` (computed lazily, same pattern as stat
decay — no cron needed). Lifecycle aging is a Phase-4 feature (§8), which
is why the schema defaults new pets to `'young'` rather than `'egg'`: v1
pets skip the pre-lifecycle stages entirely (they're "already adopted"),
and when aging lands, the pet-creation path will start new pets at `'egg'`
so the progression has somewhere to run. When aging first ships: v1 is
time-gated only. Later enhancement: also require a minimum average care
score to advance, so pure neglect doesn't still age the pet up —
deliberately deferred since it adds complexity kids may find
punishing/confusing at first.

### Hatching / birth sequence
Separate from the deferred lifecycle-aging feature above: right after
species selection, `index.html` plays a one-time animated beat sequence
(`startHatchSequence`) in place of immediate pet creation — a darkened
screen lightens onto the intact egg, the egg shakes and cracks through two
escalating stages (with a synthesized crack sound, since there's no audio
asset yet), the newborn pose appears, and the kid names their pet before
`finalizeNewPet` hands off into the normal pet screen. This only plays for
species present in the `HATCH_SEQUENCES` map; anything missing from it
falls straight through to the old immediate-creation flow — same "degrades
cleanly during art work" pattern as `POSE_FALLBACK` above, gated by map
membership rather than a fallback chain.

**Build plan per species:** four new frames beyond the standard four
lifecycle poses — `assets/<species>-egg.png`, `-egg-crack1.png`,
`-egg-crack2.png`, and `-hatch.png` — drawn to the same crop/palette/canvas
rules as the Art pipeline (§4). Once a species' four frames exist, wiring
it in is a one-line `HATCH_SEQUENCES` entry (label + the four frame paths),
the same shape as a `SPECIES_POSES` registration.

**Progress** (all six kept species are meant to get one eventually):
- ✅ T-Rex — full 4-frame set drawn and wired. Sequence timing was doubled
  2026-07-30 so kids have time to read each caption and watch the shake
  before it advances: egg hold 4.4s → "something is happening" shake-light
  3.6s → crack1 shake-light 3.2s → crack2 shake-medium 4s → hatch
  shake-big + 1s still hold before the name form appears. The shake
  keyframes themselves (`shake-light`/`-medium`/`-big`) are `infinite`
  loops, so they keep shaking for the full length of whichever stage
  applies them.
- ✅ Dragon — full 4-frame set drawn and wired (2026-07-30). Generated as a
  single 4-panel sheet (egg/crack1/crack2/hatch together, like a mini
  pose-sheet) rather than one-off calls, which is what kept scale and
  ground line consistent across frames without the drift §4 warns about
  for 8-cell sheets — four cells held fine. Departs from the T-Rex frames
  in one deliberate way: instead of a dirt nest, the egg sits on **a bed
  of glowing hot coals/embers**, matching the "fire dragon" framing. Coal
  needed its own small 6-colour sub-ramp
  (`reference/dragon-hatch-coal-palette.txt`/`.png` — deep maroon through
  red-orange to golden ember) layered on top of the locked
  `dragon-palette.txt`, since that ramp is entirely purple/blue with
  nothing warm in it.

  Building this also surfaced a real gap in the locked dragon ramp: every
  other species' palette has a near-white top value for highlights/eye-
  whites (unicorn `#fdeff5`, pegasus `#fef2d9`, hippocampus `#fbfcfb`,
  wolf `#F5F3F3`) but dragon's lightest entry was a saturated light blue
  (`#b3d5f6`). Without a true near-white to snap to, the egg's shell-shine
  and the hatchling's eye-white quantized inconsistently pixel-by-pixel
  between mismatched blues — visible as salt-and-pepper speckle rather
  than a flat highlight band, not a downsampling artifact (the pre-
  quantize box-filtered image was clean; confirmed by inspecting it
  directly). Fixed by adding `#eae4fa` (pale lavender-white, hue-shifted
  per the same one-tint-of-the-body-hue pattern every other species'
  near-white follows) as a 20th entry to `dragon-palette.txt`. Reference-
  only file, not loaded at runtime, so this didn't touch any shipped
  sprite. A first generation attempt (rejected) had a genuinely messier,
  fragmented highlight shape in the *source* art itself, not a cleanup
  bug — the regenerated source used for the shipped frames has a clean
  flat highlight blob much closer to T-Rex's, confirmed by inspecting the
  pre-quantize downsample before and after.

  Baseline registration matches the rest of the dragon family (top row
  13, bottom row 116 — `dragon.png`/`-happy`/`-sad`/`-sleep`'s own ground
  line) rather than the T-Rex family's row-117 baseline; the two species'
  egg sets aren't required to share a ground line with each other, only
  internally with their own species' poses.
  `reference/dragon-egg-reference-4x.png` holds all four hatch frames (4x)
  for attaching when generating dragon's action poses next.
- ✅ Hippocampus — **complete**: all four frames drawn, cleaned, and
  wired, plus a bubble layer unique to this species. Plan and build notes
  in
  "Hippocampus hatch sequence (planned)" below. Departs from the other
  two in that it isn't an egg at all: a mussel shell on the seabed that
  opens to reveal the baby. Everything else about the sequence — beat
  timing, shake classes, frame keys, crack sound — is unchanged; the only
  code delta was making the opening caption overridable, which is
  already in.

  `hippocampus-egg.png` (closed shell on sand) is the first frame.
  Registration used the curled-`sleep`-pose approach rather than the egg
  species' height fill, since a closed shell is wide and low: scaled by
  width with the shell at 103px and the ground line on row 116, content
  occupying rows 60–116. The composite lands at 81.2% width fill —
  the same number `trex-sleep.png` independently landed on.

  Took two generations, and the difference is worth recording because it
  is a *prompt* fix, not a cleanup fix. The first came back with a sand
  mound 1.177× the shell's width, which at the fixed 103px shell anchor
  pushed the composite to 121px — 94.5% canvas fill, 3px margins, well
  outside the 75–85% rule. Asking for "a small mound, roughly 10% wider
  than the shell, not a broad beach" brought the ratio to 1.005. Nothing
  in cleanup can fix that: the shell width is the anchor that has to hold
  across all four frames, so an oversized sand mound has nowhere to go.
  The re-roll also fixed the highlight for free — the first generation's
  specular came through as six scattered pixels in a broken diagonal (the
  fragmented-highlight shape dragon hit), where the re-roll's is a single
  connected 11px cluster needing no hand repair.

  `hippocampus-egg-crack1.png` (shell parted by a sliver, glow along the
  lip) followed, generated with frame 1 attached. Registration held
  exactly: the shell box is x 12–115, 104px wide, **identical** to frame
  1, with the top edge 2px higher — the lid lifting, which is the point.
  The raw generation drifted +2.4% in shell width and +0.6% in
  sand/shell ratio, both normalized away by scaling to the 103px anchor.

  This frame forced a fix to the crop step worth keeping. Frame 1 centred
  the *composite* bounding box on the canvas, which was fine when the
  sand and shell were the same width. Frame 2 has bubbles detached from
  the shell out to its left, so composite-centring would have shoved the
  shell right by half the bubbles' overhang — silent drift of exactly the
  kind that makes the sequence read as a pan. The crop now anchors
  explicitly on **the shell's centre → canvas x 64, and the sand's bottom
  → row 116**, with everything else free to extend where it likes. Frames
  3–4 need this even more, since an open lid and a hatchling both grow
  the composite asymmetrically.

  The bubbles were **stripped rather than kept**, per the same rule §5
  states for food and soap: items are separate files, never drawn into a
  pose. Baked-in bubbles would shake with the shell and sit frozen for
  the frame's full 3.2s, when what the sequence wants is bubbles rising
  independently of the shake. Detecting them is easy — they are connected
  components disjoint from the main silhouette — so the cleanup script
  drops every component but the largest.

  `hippocampus-egg-crack2.png` (half open, rose interior visible, the
  baby sitting in the bowl) is frame 3, and the first frame where the
  shell interior and the creature share a canvas. Registration held again
  — shell box x 11–115 against frame 1's x 12–115, sand bottom on row
  116 — even though the raw generation came back 6.9% smaller, because
  the whole composition scaled together. Content occupies rows 31–116,
  leaving 18 rows of headroom; frame 4 does not need them, since the lid
  is already the tallest element and a hatchling sitting up grows into
  the bowl's empty space rather than above the shell.

  **Check the lid's form, not just its registration.** Frame 3 took two
  generations, and the first one passed every automated check —
  registration within a pixel, palette clean, speckle 3.1% — while being
  wrong in a way none of those measure: it drew the opened upper shell as
  a *concave bowl*, a second cup facing the viewer, where frames 1–2
  establish it as a convex ridged dome. Rose interior filled 35% of the
  lid region against the correct version's 28%, because a lid turned
  inside-out shows lining that should not be visible at all. Registration
  metrics compare bounding boxes; they say nothing about whether a form
  stayed the same solid. When a frame rotates a part, compare that part's
  *shape* against the earlier frames by eye before cleaning it.

  **Never put the whole combined palette in front of the quantizer.**
  Frame 3 was first run against all 17 colours, and `#0f445c` — a locked
  ramp entry meant for the creature's dark accents — scattered
  baby-coloured pixels across the entire shell. It is not a
  minimum-distance failure; every pair in the palette is ≥60 apart. It is
  that `#0f445c` lands *on the line between* two other entries: the
  midpoint of the shell's dark band `#4c5566` and the outline `#103133`
  sits 34.7 from `#0f445c` but 43.3 from either endpoint, so every
  antialiased shell edge snapped to it. Minimum pairwise distance says
  nothing about this case — an entry can be far from all others and still
  sit inside the segment joining two of them. Dropping `#0f445c` and the
  redundant near-whites (14 colours instead of 17) cut speckle from 3.9%
  to 3.1% and confined the baby's body colours to x 63–82, the head,
  where they belong. Quantize each frame against the colours that frame
  legitimately contains, nothing more.

  The related trap: the stray-glow revert rule from frame 2 must **not**
  run on frames containing the creature. The baby's light teals are the
  same values as the glow, so the rule would erase its highlights as
  "disconnected glow". Frame 3 relies on the tightened palette instead.

  `hippocampus-hatch.png` is the reveal: the hatchling sitting up in the
  open shell, white body, teal mane and fins, tail curled beside it. The
  creature measures 38×41px against frame 3's 26×14 — 2.9× taller, and
  39% of the adult idle sprite's 104px height. That ratio is the frame's
  actual job: an earlier attempt drew the hatchling at the same 14px
  height as the peek, which read as a pretty shell with a smudge in it
  rather than as the payoff, and cut hard to a 103px idle sprite seconds
  later. Content occupies rows 22–116. Speckle is 4.6%, the highest in
  the set, from the face's fine detail; not visible at display size.

  Wired into `HATCH_SEQUENCES` with the `intro` override, so picking
  hippocampus now plays the full sequence instead of falling through to
  instant creation.

  **Anchor the mean of shell and sand, not either one.** Scaling each
  frame by the shell alone pinned the shell at exactly 103px but let the
  sand breathe 7.4px across the set (103.5 / 104.4 / 100.3 / 107.7),
  because the generator never held the sand/shell ratio constant
  (1.005 / 1.014 / 0.974 / 1.046). Scaling by the sand alone just moves
  that same 7.4px onto the shell, since frames 3–4 legitimately change
  the lid's angle and so genuinely change the shell's silhouette width.
  Anchoring the mean of the two splits the error: all four frames were
  re-cleaned that way and both now spread 3.7px. Vertically the anchor is
  unchanged — the sand's bottom row is the ground line at row 116.

  The re-clean also made the per-frame fixes algorithmic rather than
  coordinate-based, so they survive a change of scale: stray-glow revert
  and specular extension run only on frames with no creature in them, and
  the white-speckle rule is likewise gated. That gate matters — run
  unguarded on the reveal frame it deletes the hatchling's eye glints,
  which are legitimate 1px white components. The rule cannot tell a
  sparkle from a speck, so it only runs where sparkles cannot occur.

  **Bubbles are a layer, not art.** `assets/bubble-sm/-md/-lg.png` are
  hand-authored rather than generated — 5, 7 and 9px hollow rims with a
  single specular pixel, on 16×16 canvases, every pixel a locked-ramp
  entry. At that size a generated bubble would be a lumpy circle needing
  cleanup, where placing ~15 pixels by hand costs nothing and quantizes
  to nothing. The rim is `#8fd1d3` rather than the softer `#a4e4de`: the
  stage's own background is `#cdeaf7`, and `#a4e4de` sits only 48 from it
  in RGB where `#8fd1d3` sits 76, so the softer value nearly vanished
  against the water.

  They are spawned as elements into `.hatch-bubbles`, a sibling of the
  sprite inside `.hatch-stage`, for the reason §5 keeps items out of
  poses: the shake classes transform `.hatch-sprite`, and bubbles drawn
  into a frame would shake with the shell instead of rising past it —
  besides sitting frozen for the 3–4s a frame is on screen. The layer
  paints above the sprite (bubbles pass in front of the lid) but below
  `.hatch-dark`, so nothing shows during the dark beat. Each bubble is
  sized from the sprite's *rendered* width — `16 × (spriteWidth / 128)` —
  so a bubble pixel matches a shell pixel at any stage size; a fixed 16px
  renders visibly finer than the art it sits next to.

  Rates follow the beats: one every 700ms from the first crack, 430ms
  while the shell strains open, then settling to 1000ms after the reveal.
  `bubbles: true` is opt-in per species — T-Rex and dragon hatch on a
  dirt nest and a bed of coals, where rising bubbles would be nonsense.

  `reference/hippocampus-egg-reference-4x.png` holds all four frames at
  4× (2048×512). Unlike
  dragon's hatch reference it sits on flat `#FF00FF` rather than a dark
  backdrop, so it doubles as a demonstration of the background the next
  generation must produce.
- ⬜ Unicorn, pegasus, wolf — no hatch art yet; picking these species
  still skips straight to instant pet creation. Remaining work is
  art-only (four frames each), no further code changes needed beyond
  their `HATCH_SEQUENCES` entry.

#### Hippocampus hatch sequence (planned)

A seahorse-dragon shouldn't crack out of a bird's egg, so the hippocampus
sequence swaps the shell for a **clam/oyster resting on the seabed**,
the same way dragon swapped the dirt nest for hot coals — the beat
structure, timing, shake classes, and frame *keys* are unchanged, only
what's drawn inside them and what the captions say.

**Beat → frame mapping.** Three story beats over four frames; the middle
beat gets two frames because "it's opening" is the part that needs to
read as motion in a slideshow of stills:

| Key | File | Beat | What's drawn | Shake |
|---|---|---|---|---|
| `egg` | `hippocampus-egg.png` | "it's a clam" | Clam sealed shut, ridged shell, sitting in a low mound of sand | none → `shake-light` |
| `crack1` | `hippocampus-egg-crack1.png` | "it's opening" (1) | Shell parted a sliver; a thin flat band of glow and 2–3 bubbles escaping the gap | `shake-light` |
| `crack2` | `hippocampus-egg-crack2.png` | "it's opening" (2) | Shell half open, hinged back ~45°; nacre interior visible; two eyes and the snout tip peeking over the lower lip | `shake-medium` |
| `hatch` | `hippocampus-hatch.png` | "a baby hippocampus!" | Shell wide open, baby sitting up in the lower valve — the direct analogue of `dragon-hatch.png`'s hatchling-in-eggshell | `shake-big` → still |

Keep the `-egg` filenames even though nothing is an egg. The names track
the *slot* (they're what every other species uses, and the `frames` keys
in `HATCH_SEQUENCES` are `egg`/`crack1`/`crack2`/`hatch` regardless), and
the path values are free-form strings, so this is a naming-consistency
call rather than a load-bearing one.

**Registration — the one real trap here.** The other two species' first
three frames are a tall egg that fills the canvas by height (rows 13–116).
A closed clam is wide and low, so it can't be height-filled without
becoming a beachball. Register it like the curled `sleep` poses instead:

- Fix the **shell width** once at ~103px (≈81% width fill, the number
  `hippocampus-sleep.png` already landed on) and hold that width across
  all four frames. Register on the *shell's* bounding box, not the
  composite — if the frames are each fit to the canvas independently, the
  shell visibly grows as the baby adds height, and the sequence reads as
  a zoom instead of an opening.
- Bottom row **116** on every frame, matching the four existing
  hippocampus poses (per the dragon precedent, an egg set only has to
  share a ground line with its own species).
- Total height grows frame to frame as the lid swings back, capped at top
  row **13** on `hatch` so the tallest frame still fits the shared
  104px band. Closed shell + sand ≈ 45px tall; budget the rest for the
  lid and the baby.

**Palette.** The creature stays on the locked 10-colour
`reference/hippocampus-palette.txt` — the baby has to match
`hippocampus.png` because the pet screen cuts straight to that idle
sprite seconds later. The shell and sand don't exist on that ramp (it's
entirely teal/aqua plus near-whites), so they get a sub-ramp layered on
top exactly as dragon's coals did:
`reference/hippocampus-hatch-shell-palette.txt`/`.png`, 7 entries.

The shell is **grey-blue outside, rose-pearl inside** — mussel rather
than oyster. That split is deliberate and load-bearing, not decoration:
the cool exterior gives the shell its own identity against a teal
creature, while the warm interior is what the baby is actually seen
against in the reveal frame.

| Hex | L\* | C\* | h | Role |
|---|---|---|---|---|
| `#4c5566` | 36.0 | 11.0 | 274 | Shell ridge shadow (exterior darkest) |
| `#88674b` | 46.2 | 22.9 | 66 | Sand shadow |
| `#707f91` | 52.6 | 11.6 | 263 | Shell exterior mid |
| `#b4915c` | 62.3 | 33.6 | 79 | Sand light |
| `#99a1b1` | 66.1 | 9.3 | 273 | Shell exterior light / rim |
| `#da9690` | 68.6 | 28.2 | 29 | Nacre shadow (inner rim) |
| `#f9cbc4` | 85.4 | 18.0 | 32 | Nacre lit (interior surface) |

Plus two **reused** locked entries rather than new ones: `#fbfcfb` as the
pearl highlight and `#103133` as the outline for shell and sand both, so
the whole frame keeps the species' single hue-shifted outline.

These values were picked against the quantizer's actual behaviour, not by
eye. Cleanup step 5 snaps every pixel to its nearest palette entry by RGB
distance, so what matters is the *minimum distance in the combined
17-colour palette* — two entries at distance d put a decision boundary at
d/2, and source pixels near that boundary flip between them, which is
what speckle is. The shipped benchmark is dragon's coal ramp at 63.4;
this ramp holds **59.7**, its tightest pair being the shell rim
`#99a1b1` against the locked light teal `#8fd1d3`. Four things shaped it:

- **No second near-white.** A pale nacre highlight is the natural choice
  and it is a trap: at L\* ≈ 90 it lands ~45 from the locked
  `#eef6f5`/`#f5fbf9`/`#fbfcfb` cluster, under half the working spacing,
  reproducing dragon's speckle exactly with warm and cool whites
  alternating pixel to pixel. Reusing `#fbfcfb` removes the failure mode
  instead of tuning around it.
- **The interior must out-light the baby.** Nacre lit sits at L\* 85.4
  against the baby's light teal at 79.7 and body mid at 73.0, so the
  hatchling reads as a darker shape on a bright ground. An earlier
  candidate at L\* 73.6 was almost exactly the body mid's lightness — the
  silhouette would have depended on hue alone and gone muddy at 128px,
  and worse in greyscale or for a colourblind kid.
- **A cool exterior costs separation, and 59.7 is the ceiling.** A
  rose-grey exterior scored 64.1; moving the shell to the cool half of
  the wheel gives up ~5 because three low-chroma blue-greys have to
  thread between the locked ramp's dark blue-teal (`#0f445c`, L\* 26.8)
  and its light teals (L\* 73–80). Below ~L\* 34 the ridge shadow closes
  on `#0f445c`; above ~L\* 68 the rim closes on `#8fd1d3`. L\* 36/53/66
  is roughly the widest spacing that fits, and compressing to two
  exterior values instead of three does not help — it just moves the
  collision. 59.7 still leaves ~4× the per-channel noise the box-filter
  downsample produces in flat regions, and the one tight pair never forms
  a soft boundary in the art: the baby always carries its `#103133`
  outline between its body and the shell rim, so there is no gradient
  across that edge for the quantizer to flicker along.
- **Don't let the optimizer make it grey.** Left free, the search drove
  the exterior to C\* 4–5, which maximizes distance but reads as neutral
  grey with no blue in it at all. The values above hold a C\* ≥ 9 floor
  so the blue is actually visible; that floor is what pulled the minimum
  from 66.2 down to 59.7. Similarly, the hue is pinned near 265–275: left
  looser it drifted to ~285, which is periwinkle rather than grey-blue.

Everything else stays on-ramp: build the glow leaking from the gap out of
the locked `#c6eee8`/`#a4e4de` as flat bands, never a gradient, and the
bubbles from `#fbfcfb` and `#c6eee8`.

One aesthetic note the numbers don't cover: hold the interior's chroma at
C\* ≤ 28 (dusty rose, not candy pink). Unicorn's locked ramp is 20 shades
of saturated pink, and a high-chroma interior would read as unicorn parts
rather than shell.

**Generation.** One 4-panel sheet in a single call, as dragon did — four
cells held registration fine where eight drifted. Attach both
`reference/hippocampus-reference-4x.png` (so the baby reads as the same
character) and the two palette PNGs. Ask for: flat cel-shaded pixel art,
no anti-aliasing, hue-shifted dark-teal outline, flat `#FF00FF`
background, no props/grass/shadow beyond the sand the clam sits in, and
the same shell size in all four cells. Baby proportions are newborn —
bigger head, shorter snout, stubbier fins, tail curled in the shell — and
facing the same 3/4-left direction as `hippocampus.png` so the cut to the
pet screen isn't a flip.

**Cleanup.** Standard pipeline, with the one landmine this species has
already hit once: **key magenta to alpha *before* cropping**, or the
margin inside the crop box quantizes into a visible rectangle (see
`hippocampus-sad.png` above). Then quantize onto locked ramp + shell
sub-ramp, ghost-overlay each frame against the previous to confirm the
shell hasn't moved or resized, and save
`reference/hippocampus-egg-reference-4x.png` with all four at 4×.

**Captions — one line, already wired.** Only the opening caption names
the container (`"What's this? A " + label + " egg..."`); every later beat
is either container-neutral ("Something is happening!") or says "the baby
`<label>`", all of which are true of an oyster. So the sequence needs a
single override, not a caption system: `startHatchSequence` now reads an
optional `intro` string off the `HATCH_SEQUENCES` entry and falls back to
the egg wording when it's absent, leaving T-Rex and dragon untouched.
Hippocampus's entry sets:

> `intro: 'Look, a giant oyster. I wonder what’s inside?'`

(Typographic apostrophe, as the existing captions use — a straight one
would close the single-quoted string.)

That is the whole code delta — with it in place, wiring the art is still
the promised one-entry registration. The synthesized `playCrackSound()`
noise burst reads fine as shell-on-shell knock and stays as-is.

### Species selection
Species is picked once, at first launch, and then locked for the life of
that pet — `index.html` shows a dedicated "Choose your pet!" screen
when no species is stored yet, and once one is picked the picker is hidden
and the choice is written to `localStorage` (and later, `pets.species` in
D1) permanently. Kids don't get a species switcher in the main app.

This is deliberately not user-switchable yet. A later phase can add a
gated way to change species post-creation (e.g., unlocked via a care
streak or points threshold, mirroring the accessory-unlock mechanic
below) rather than leaving it freely swappable, which would undercut the
"raising one pet" framing. The one-time-pick guard in `index.html` is a
single `if (state.species) return;` check in the species button handler —
swap that for the real gating condition when this ships.

### Accessories
Equip via `pet_equipped_items`. Rendered as image overlays positioned per
the anchor config above. Acquired via care streaks, sibling gifting (reuses
the gifting mechanic below), or a simple points-based in-app shop — no real
money involved anywhere.

### Change location
The `pet_current_location` row for a pet (`pet_id → location_id`, §3
deferred tables) swaps the background image behind the pet. Cheapest
feature here — no per-species art, no anchor math, just an image swap.
Unlockable the same way as accessories, or free to switch among
whatever's unlocked. Kept as a separate one-row-per-pet table rather
than a column on `pets` so the location catalog can grow without a
migration touching the hot `pets` row.

### Color/variation at species selection
At creation, alongside picking species, the kid picks a color variant
(stored as a named preset). Shipped on `main` — the notes below are the
design rationale, not a plan.

Originally specced as a CSS `hue-rotate` filter over the base sprite. That
no longer held once §4 locked a shared palette: hue-rotating by an
arbitrary angle produced colors that were by definition off-palette, and
it discolored incidental parts (teeth, eye whites, mouth interior) along
with the intended body color — visible on the pre-remap unicorn, whose
gold hooves and horn shifted with its mane.

Instead, each species ships **2–3 hand-picked palette swaps**: a small map
of `{ source palette index → replacement palette index }` per variant,
applied once per sprite at load time on an offscreen canvas and cached.
Bounded work (2–3 short mappings per species, not an image per color ×
species), stays on palette, and leaves the parts that shouldn't change
alone. Because it's a palette remap rather than a filter, it applies
uniformly across every pose without per-pose tuning.

### Simultaneous own + sibling pet during visits
The one item here with real backend weight. Visiting a sibling means
rendering two pets side-by-side: the visitor's own pet (local/cached) and
the host's pet fetched from the server. Two implementation options:
- **v1 (recommended to start, matches §2): polling.** Fetch the host pet's
  current state/location/accessories from `/api/pets/:id` every few
  seconds while visiting. Simple to build, no WebSocket or Durable Object
  infrastructure needed, "live enough" for kids taking turns caring for
  each other's pets.
- **v2: Durable Objects + WebSocket push.** A per-pet DO holds open
  WebSocket connections and broadcasts state changes instantly to any
  connected visitor — needed if we want truly simultaneous co-visit (both
  kids acting on the host pet at once, or a synchronous two-pet mini-game
  later). This is the deferred item called out in §2.

Recommendation matches §2: build polling first. It satisfies "see both
pets on screen at once" fully, and upgrading to DO + WebSocket push later
is a transport change, not a data-model change.

## 6. Social / visiting features (recap)

Since all four kids are one family, "friends" is just the sibling roster —
gated by a family passcode, no add-friend flow. Recommended v1 slice:
read-only visiting + once-a-day helper actions (feed/play on a sibling's
pet) + gifting + presence indicator. Live co-visit and any mini-games stay
deferred until the core loop and the polling-based visit view are solid.

## 7. Identity/auth

No email/password. Adapted from `NoliCommoveri/Star-homeschool`
`docs/parent-sync-spec.md §5–6.5`.

**Family creation** — one-time, gated by the `SIGNUP_SECRET` env var (a
long random string you paste into the Cloudflare dashboard's Variables &
Secrets tab, §9 Step 7). The setup flow POSTs to `/api/family` with
`{signupSecret, deviceId, label}` and gets back a device token. This is
what keeps the deployment serving exactly one family — without it, anyone
who could hit `/api/family` could spawn their own family in your D1.

**Additional device pairing** — an already-paired device calls
`/api/pairing-code` to mint a short-lived 6-char code (5-min TTL,
single-use); the new device POSTs to `/api/pair` with the code and
receives its own device token. No secret involved — the code *is* the
secret, briefly. Same mechanism a kid uses to add their second tablet or
a friend joining later (see also §5 "friends" note).

**Kid rows** are created via `POST /api/kids` (auth'd, family-scoped from
the device token) — the setup UI on the bootstrap device makes one row
per sibling up front (display_name + avatar), and additional kids can be
added later from any already-paired device in the family. Kid rows are
distinct from device rows: a kid can exist before any device has claimed
them, and a re-paired tablet can re-claim the same kid without recreating
the row.

**Per-request auth** — every non-`/api/family` and non-`/api/pair`
request carries `Authorization: Bearer <device-token>`. The Worker
hashes the token, looks it up in `devices`, checks `revoked = 0`, and
updates `last_seen`. A device inherits its family membership from its
row; a device claims a kid identity by POSTing to
`/api/kids/:id/claim` at first-run — this is where the kid picks their
avatar (or confirms one already set at kid-creation time). PIN is
client-side only in v1 — the server treats device tokens as
authoritative — but the schema keeps `pin_hash` so a "must PIN before you
can act on your pet" gate can land later without a migration. Plain
SHA-256 of a 4-digit PIN (even with a per-kid salt) is trivially
brute-forced — before that gate is server-enforced, `pin_hash` must
switch to argon2/scrypt with a real work factor. Schema-comment reminder
lives in §9 Step 5.

**Family-scope check on every write** — the Worker looks up the target
pet's `kid_id → family_id` and confirms it matches the acting device's
`family_id`. For helper actions on a sibling's pet, it additionally
enforces the once-a-day rate limit via `helper_action_usage`
(`INSERT OR ABORT` on the composite PK does the work, §3).

## 8. Build phases

1. **MVP (done)**: single pet, stats/mood engine, species picker, local
   persistence — this is `index.html` today.
2. **Real art (in progress)**: pose-swapping renderer and fallback chain
   are built, and six species (T-Rex, unicorn, dragon, hippocampus,
   pegasus, wolf) each have all four required poses (idle/happy/sad/sleep),
   wired into `SPECIES_POSES` and the picker; palette-remap color variants
   exist for five of the six (all but wolf). Phoenix/griffin/blob are
   dropped from the picker, `SPECIES_POSES`, and (phoenix/griffin) the
   asset files. Per-species feed/toy items (two each) are drawn and wired
   into `SPECIES_ITEMS` for five of the six species (all but wolf, which is
   unused); clean uses one shared bubble sprite across all species. The item
   approach (lower-left entry, shrinking to landing scale at a fixed
   front-of-pet anchor) is built and needs no further art — remaining there
   is just tuning that anchor per species. Still missing: the four action
   poses (eat, eat-chew, play, bath) per species, and the reaction loops
   (shake/bounce CSS, plus the eat/eat-chew alternation) that need them.
3. **Backend + Social v1 (in progress)**: Cloudflare Workers with
   `[assets]` binding + D1, deployed and live at `critteria.immotus.app`
   (§9 Steps 1–8 done). `SIGNUP_SECRET`-gated family creation is verified
   working against the real deployment; the pairing-code round-trip is
   still unconfirmed against the live deployment (§9 Step 9). The
   frontend sync layer (§9 Step 10) is now built: `index.html` has a
   family-setup screen (first-tablet secret or pairing code), a kid
   picker (pick/add + claim), server-synced pet state polled every ~5 s
   while foregrounded with `localStorage` as warm cache/offline
   fallback, and a "Visit a sibling" screen with Feed/Play helper-action
   buttons wired to the once-a-day-limited endpoint. A device with no
   family token still plays entirely offline exactly as before (species
   picker → local decay/actions), and a device that already has a
   locally-raised pet is never force-migrated — connecting is an opt-in
   🔗 tap that upload-creates its pet server-side. `PATCH /api/pets/:id`
   now persists color-variant swatch changes server-side (previously the
   swatch click only updated local state, so the next 5 s poll silently
   reverted it — fixed). Not yet wired in the UI: gifting
   (`POST /api/pets/:id/gift` exists server-side, no frontend for it)
   and the presence badge from `pet_events`; renaming stays device-local
   since there's no rename endpoint yet. Concrete deploy recipe and
   every gotcha we've hit are in §9.
4. **End-state features**: lifecycles/aging, accessories, locations
   (color variants already landed on `main` as palette-remap work).
5. **Nice-to-haves**: Durable Objects + WebSocket for live co-visit
   (transport upgrade over the polling shape in §2, no data-model
   change), push notifications, mini-games, guestbook/stickers, family
   (non-competitive) garden score.

## 9. Backend deployment (Cloudflare Workers + D1)

Written to be followed without prior Cloudflare knowledge. Adapted from
`NoliCommoveri/Star-homeschool` `docs/parent-sync-spec.md §12` — that
document is the canonical explanation of *why* each step is shaped the
way it is; this section is the concrete recipe for Critteria and the
gotchas we've either hit ourselves or inherited from star.

**Cloudflare moves its dashboard around.** Pages has been folded into
Workers entirely, so "Pages project" in older writing (including earlier
drafts of this file) now just means "Worker." Each step below states
*what you are accomplishing* and matches on that goal, not on dashboard
menu names that might have changed by the time you read this.

This touches the repo beyond app files: `wrangler.toml`, `.assetsignore`,
`schema.sql`, `functions/api/*`, and a committed `dist/worker/index.js`
build output. None of it changes behavior when the API isn't reachable —
the app runs on `localStorage` alone, identical to today.

### File layout (target)

```
index.html               Frontend PWA (existing).
assets/                  Sprites, palettes, reference art (existing).
sw.js                    (planned) Service worker for PWA install/offline.
wrangler.toml            Worker config: [assets] + [[d1_databases]].
.assetsignore            What NOT to upload as public static assets.
schema.sql               D1 schema (see Step 5 for the full contents).
functions/api/           Pages-Functions-style API handlers:
  family.js              POST — create family (SIGNUP_SECRET-gated).
  pairing-code.js        POST — mint a pairing code (auth'd).
  pair.js                POST — redeem a pairing code, get device token.
  kids.js                GET, POST — list/create kids (family-scoped
                         from the device token; the setup UI on the
                         bootstrap device seeds one row per sibling).
  kids/[id].js           PATCH — update kid profile (display_name,
                         avatar). PIN edits go through here too once
                         `pin_hash` is upgraded per §7.
  kids/[id]/claim.js     POST — bind current device to a kid.
  devices.js             GET — list family's devices.
  devices/[id].js        DELETE — revoke a device.
  pets.js                GET, POST — list/create pets in family.
  pets/[id].js           GET — server-computed pet state (lazy decay).
                         PATCH — update color_variant (own pet only).
  pets/[id]/action.js    POST — feed/play/clean/sleep/wake (own pet
                         always; sibling pet gated by helper_action_usage).
  pets/[id]/gift.js      POST — send a gift (writes pet_events).
  pets/[id]/events.js    GET — recent events (presence + short history).
dist/worker/index.js     Bundled Worker script. Committed because
                         Cloudflare's git-connected deploy does NOT run
                         the bundler on its side.
```

### Step 1 — Cloudflare account

Sign up at dash.cloudflare.com. Free plan, no credit card. You do not
need to buy a domain or move DNS anywhere.

*Checkpoint:* you can see a dashboard with **Workers & Pages** in the
sidebar.

### Step 2 — Connect the repo (done)

Workers & Pages → Create → connect to Git → authorize GitHub → pick
`NoliCommoveri/Critteria`, branch `main`. There is no separate "Pages"
option anymore — connecting a repo always creates a Worker.

Until `wrangler.toml` exists (Step 6 below), Cloudflare deploys the repo
as static-assets-only (no server code), which is fine — that's the state
Critteria is in as of this section being written. All of a git-connected
Worker's config (what to serve as static assets, what script to run,
what it's bound to) comes from `wrangler.toml`; the dashboard has no
build-output-directory field or D1-binding UI for a git-connected
project.

*Checkpoint:* the site at `critteria.<your-subdomain>.workers.dev`
(dashboard → **Visit**) loads `index.html` and the pet works exactly as
it does on localhost. **Test this before continuing** — this is the
moment hosting moves off whatever host it was on.

### Step 3 — Custom domain (done)

Worker → Domains & Routes → **Add** → `critteria.immotus.app`. If your
domain is already in Cloudflare, DNS is one click; otherwise the
dashboard walks you through the CNAME.

The site loads at `critteria.immotus.app` today via this route, serving
the repo as static assets. A GitHub Pages `CNAME` file briefly lived at
the repo root for a never-realized GH Pages hosting path — it was
removed in commit `5d2f157`, along with the README section that
described GH Pages as the host. GitHub's "DNS check successful" +
"Enforce HTTPS unavailable" state on that repo is inert zombie config
and can be ignored (or GH Pages disabled outright in repo Settings).

### Step 4 — Create the D1 database (done)

Workers & Pages → D1 → **Create database** → name it `critteria`.

CLI equivalent: `npx wrangler d1 create critteria`

*Checkpoint:* the database appears in the D1 list with a database ID.
Copy the ID for Step 6.

### Step 5 — Apply the schema (done)

Commit the `CREATE TABLE` statements below to the repo as `schema.sql`,
then apply them to the remote D1:

```
npx wrangler d1 execute critteria --remote --file=./schema.sql
```

**Don't paste `schema.sql` into the D1 Console tab in the dashboard.** It
looks plausible but silently mishandles multi-statement, commented SQL —
pasting the whole file gives a "request malformed" error, and stripping
the comments just moves the failure to "incomplete input." The CLI
command above sends the file to D1's real batch-exec endpoint and works
correctly. If you truly have no CLI, the Console can work one statement
at a time (no comments, one `CREATE TABLE`/`CREATE INDEX` per paste),
but there is no reason to do it that way.

**We hit this ourselves: the Console's one-statement-at-a-time path let a
column silently drift from schema.sql.** The `devices` table went live
without its `label` column, which didn't surface as an error at table-
creation time — it surfaced later as a `D1_ERROR: table devices has no
column named label` when `/api/family` tried to insert a row (§9 Step 9).
Fix in place was `ALTER TABLE devices ADD COLUMN label TEXT;`, since the
table was still empty. If you go the one-statement-at-a-time route, verify
each table immediately after creating it with
`PRAGMA table_info(<table>);` against the column list below, rather than
trusting the `CREATE TABLE` succeeded exactly as pasted — the CLI's
single `--file` apply doesn't have this failure mode, which is the real
reason to prefer it.

**The `--remote` flag matters.** Without it, wrangler writes to a local
dev copy and the real database stays empty — a genuinely confusing
failure, because every command appears to succeed.

*Checkpoint:* `SELECT name FROM sqlite_master WHERE type='table';` in
the D1 Console lists `families`, `kids`, `devices`, `pairing_codes`,
`pets`, `pet_events`, `helper_action_usage`, plus the four Phase-5
catalog tables (Cloudflare's own `_cf_KV` table also appears — that
one's not yours, ignore it).

Note the `PRAGMA foreign_keys = ON` at the top of the schema: D1
supports FKs but does not enforce them by default. **The pragma applies
per-connection, and a git-deployed Worker does not run it at request
time** — so the schema's reliance on FK enforcement is currently
theoretical, not actually active in production. Real enforcement would
need `PRAGMA foreign_keys = ON` run at the top of the request handler,
not just once at schema setup. Track carefully.

#### schema.sql

```sql
-- Critteria backend schema (see SPEC.md §3, §9).
-- Apply with: npx wrangler d1 execute critteria --remote --file=./schema.sql

PRAGMA foreign_keys = ON;

CREATE TABLE families (
  id          TEXT PRIMARY KEY,       -- 128-bit random hex, never derived
  created_at  INTEGER NOT NULL
);

CREATE TABLE kids (
  id           TEXT PRIMARY KEY,
  family_id    TEXT NOT NULL REFERENCES families(id),
  display_name TEXT NOT NULL,
  avatar       TEXT,                  -- preset key or emoji
  -- PIN is client-side only in v1 (§7). Plain SHA-256 of a 4-digit PIN
  -- with a per-kid salt is trivially brute-forced (10 000 candidates,
  -- no work factor) — before this column ever gates a server-side
  -- action, migrate to argon2/scrypt with a real work factor. Nullable
  -- until then; do not read it as authoritative.
  pin_hash     TEXT,
  created_at   INTEGER NOT NULL
);
CREATE INDEX idx_kids_family ON kids(family_id);

CREATE TABLE devices (
  id          TEXT PRIMARY KEY,       -- client-generated UUID; stable across re-pairing
  family_id   TEXT NOT NULL REFERENCES families(id),
  kid_id      TEXT REFERENCES kids(id),  -- null until claimed
  token_hash  TEXT NOT NULL UNIQUE,   -- SHA-256; raw token never stored; rotates
  label       TEXT,                   -- e.g. "Ana's tablet"
  created_at  INTEGER NOT NULL,
  last_seen   INTEGER,
  revoked     INTEGER NOT NULL DEFAULT 0
);
CREATE INDEX idx_devices_token ON devices(token_hash);

CREATE TABLE pairing_codes (
  code_hash   TEXT PRIMARY KEY,       -- SHA-256 of 6-char code
  family_id   TEXT NOT NULL REFERENCES families(id),
  expires_at  INTEGER NOT NULL,       -- 5-min TTL
  used_at     INTEGER                 -- non-null once redeemed; single use
);

CREATE TABLE pets (
  id            TEXT PRIMARY KEY,
  kid_id        TEXT NOT NULL UNIQUE REFERENCES kids(id),  -- one pet per kid
  species       TEXT NOT NULL,
  color_variant TEXT,
  name          TEXT,
  stage         TEXT NOT NULL DEFAULT 'young',
  created_at    INTEGER NOT NULL,
  hunger        REAL NOT NULL DEFAULT 80,
  happiness     REAL NOT NULL DEFAULT 80,
  energy        REAL NOT NULL DEFAULT 80,
  cleanliness   REAL NOT NULL DEFAULT 80,
  sleeping      INTEGER NOT NULL DEFAULT 0,
  last_updated  INTEGER NOT NULL      -- server epoch ms; decay applied lazily
);
CREATE INDEX idx_pets_kid ON pets(kid_id);

CREATE TABLE pet_events (
  id            INTEGER PRIMARY KEY AUTOINCREMENT,
  pet_id        TEXT NOT NULL REFERENCES pets(id),
  actor_kid_id  TEXT NOT NULL REFERENCES kids(id),
  action        TEXT NOT NULL,        -- 'feed'|'play'|'clean'|'sleep'|'wake'|'visit'|'gift'
  occurred_at   INTEGER NOT NULL,
  detail        TEXT                  -- optional JSON
);
CREATE INDEX idx_pet_events_pet ON pet_events(pet_id, occurred_at DESC);

-- Composite PK enforces once-a-day-per-action-per-visitor-per-host-pet:
-- an INSERT OR ABORT either succeeds or hits the daily rate limit with
-- no race window. Do NOT use INSERT OR IGNORE — it would silently
-- succeed on the duplicate and let the helper action run.
CREATE TABLE helper_action_usage (
  visitor_kid_id  TEXT NOT NULL REFERENCES kids(id),
  host_pet_id     TEXT NOT NULL REFERENCES pets(id),
  action          TEXT NOT NULL,
  utc_day         TEXT NOT NULL,      -- 'YYYY-MM-DD' (UTC)
  used_at         INTEGER NOT NULL,
  PRIMARY KEY (visitor_kid_id, host_pet_id, action, utc_day)
);

-- Deferred to Phase 5 (§8). Kept here so a future migration doesn't
-- need to reshape any of the above.
CREATE TABLE items (
  id              TEXT PRIMARY KEY,
  slot            TEXT NOT NULL,      -- 'head'|'neck'|'back'|...
  name            TEXT NOT NULL,
  image_asset     TEXT NOT NULL,
  species_compat  TEXT NOT NULL,      -- JSON array of species keys
  unlock_type     TEXT NOT NULL       -- 'points'|'streak'|'gift'
);

CREATE TABLE pet_equipped_items (
  pet_id   TEXT NOT NULL REFERENCES pets(id),
  slot     TEXT NOT NULL,
  item_id  TEXT NOT NULL REFERENCES items(id),
  PRIMARY KEY (pet_id, slot)
);

CREATE TABLE locations (
  id           TEXT PRIMARY KEY,
  name         TEXT NOT NULL,
  image_asset  TEXT NOT NULL,
  unlock_type  TEXT NOT NULL
);

CREATE TABLE pet_current_location (
  pet_id       TEXT PRIMARY KEY REFERENCES pets(id),
  location_id  TEXT NOT NULL REFERENCES locations(id)
);
```

### Step 6 — Bind D1 in wrangler.toml (done)

**The dashboard's Bindings tab does not work for a git-connected Worker.**
Any binding added through the UI silently fails to persist because the
real source of truth is `wrangler.toml` and the dashboard won't let the
two drift. (You'll know you're in this state if Settings → Variables and
secrets says "Variables cannot be added to a Worker that only has static
assets," or if the Bindings tab accepts a new D1 binding but shows "No
connected bindings" again right after.)

Instead, commit `wrangler.toml` to the repo root:

```toml
name = "critteria"
main = "./dist/worker/index.js"
compatibility_date = "<today's date, YYYY-MM-DD>"

[assets]
directory = "./"
binding = "ASSETS"

[[d1_databases]]
binding = "DB"
database_name = "critteria"
database_id = "<your D1 database's ID from Step 4>"
```

Find the database ID on the D1 database's own Overview page in the
dashboard, or via `npx wrangler d1 list`. `main` points at a build
output that doesn't exist until Step 8 — that's expected.

*In code this becomes* `env.DB` inside function handlers, and
`env.ASSETS` for static-file fallback.

### Step 7 — Set the signup secret (done)

Worker → Settings → **Variables and secrets** → add `SIGNUP_SECRET`,
type **Secret**, value = a long random string you generate:

```
# macOS/Linux
openssl rand -hex 32
```

```powershell
# Windows PowerShell
-join ((48..57) + (65..90) + (97..122) | Get-Random -Count 48 | % { [char]$_ })
```

Unlike the D1 binding, secrets genuinely work from the dashboard — they
are stored separately from `wrangler.toml` (correctly: never commit a
secret value to git) and aren't subject to the same config-drift lock.
Just the one environment; no Production/Preview split to worry about
for a git-connected Worker like this one.

This is the §7 gate that keeps your instance serving exactly one
family. Once saved, the value is **not viewable again** — copy it into
a password manager the moment you create it.

### Step 8 — Write the functions, bundle, and commit (done)

Create `functions/api/` per the layout above, one file per endpoint,
using Pages-Functions file-based-routing shape:

```js
// functions/api/pets/[id].js
export async function onRequestGet({ request, params, env }) {
  const token = (request.headers.get('Authorization') || '')
    .replace('Bearer ', '');
  // ...hash the token, verify against env.DB, load pet, apply lazy
  //    decay, return current state as JSON...
  return Response.json({ pet: { /* ... */ } });
}
```

Then bundle:

```
npx wrangler pages functions build --outdir=dist/worker
```

**This file-based routing is a Pages-only convention — a plain Worker
(which is what git-connected deploys produce now) does not understand
`functions/` at all** and will not serve any of it. The bundler
collapses `functions/api/*.js` into one `dist/worker/index.js` that does
its own internal routing and falls back to `env.ASSETS.fetch(request)`
for anything it doesn't handle — which is exactly what `wrangler.toml`'s
`main` points at.

**Commit `dist/worker/index.js`.** Cloudflare's build side does not
regenerate this. It has to be rebuilt and committed by hand after every
change under `functions/`, before you push. Forgetting = every endpoint
404s on the next deploy.

Also commit `.assetsignore` in the repo root, or the `[assets] directory
= "./"` from Step 6 will upload things that were never meant to be
public:

```
.git/
.wrangler/
functions/
dist/
schema.sql
wrangler.toml
.assetsignore
README.md
SPEC.md
node_modules/
package.json
package-lock.json
```

(`docs/` isn't listed because the repo has no `docs/` directory today —
add the line if that changes.)

**The `.git/` line is not optional.** Wrangler's default ignore list
only covers `.assetsignore`, `_redirects`, and `_headers` — it does
*not* skip `.git` or `node_modules` on its own. Without `.git/` here,
the entire commit history — every past version of every file — gets
uploaded as publicly downloadable static assets alongside the site.

*Gotcha:* **bindings and secrets only take effect on a deploy made
after they were added.** If you added the D1 binding or `SIGNUP_SECRET`
to an existing deployment, push a new commit (or hit Retry deployment)
or `env.DB` / `env.SIGNUP_SECRET` will still be undefined at request
time.

### Step 9 — Deploy and verify (partially done)

`git push origin main`. Watch the deployment go green in the CF
dashboard, then verify the API is alive before wiring the frontend:

**Confirmed working on the live deployment:** the `/api/kids` no-token
check, and a real `/api/family` call that returned a device token (after
the Step 5 schema-drift gotcha above was fixed). Not yet confirmed: the
wrong-secret rejection, and the pairing-code round-trip. Whoever picks
this back up should run those before calling Step 9 fully done.

```
curl -i https://critteria.immotus.app/api/kids
# expect 401 — no token. A 404 means routing never reached the Worker
#   (check that functions/ was actually rebuilt into dist/worker/index.js
#   per Step 8, and if it still 404s, that [assets].run_worker_first
#   isn't needed for your setup).
# A 500 usually means the D1 binding didn't apply (Step 6, or the Step 8
#   deploy-after-binding gotcha).
```

Then create your family — the one call that needs `SIGNUP_SECRET`:

```
curl -X POST https://critteria.immotus.app/api/family \
  -H 'Content-Type: application/json' \
  -d '{"signupSecret":"<the secret>","deviceId":"'"$(uuidgen)"'","label":"first tablet"}'
# expect a JSON body with a device token. Save it, this is your bootstrap
# admin device.
```

**On Windows, use `curl.exe` explicitly** — PowerShell's built-in `curl`
is an alias for `Invoke-WebRequest` and takes different flags, so
`-X`/`-H`/`-d` either error or mean something else. Also swap in a
fixed UUID for `deviceId`, since `uuidgen` isn't a Windows command.
PowerShell equivalent:

```powershell
curl.exe -X POST https://critteria.immotus.app/api/family `
  -H "Content-Type: application/json" `
  -d '{\"signupSecret\":\"<the secret>\",\"deviceId\":\"00000000-0000-0000-0000-000000000001\",\"label\":\"first tablet\"}'
```

Note the backslash-escaped double quotes inside `-d`'s single-quoted
value — PowerShell's quote handling for `curl.exe` arguments needs
JSON's inner `"` escaped even though it looks like it shouldn't. The
backtick at end of line is PowerShell's line continuation, the
equivalent of Bash's trailing `\`.

Then confirm the §7 gate actually holds — a wrong secret must fail:

```
curl -X POST https://critteria.immotus.app/api/family \
  -H 'Content-Type: application/json' \
  -d '{"signupSecret":"wrong","deviceId":"x","label":"x"}'
# expect 401/403. A 200 here means your instance is open to the world —
# stop and audit /api/family before continuing.
```

Then exercise the pairing round-trip — the mechanism every second-onwards
tablet uses (§7). Use the bootstrap device's token from the family-
creation step:

```
# Mint a pairing code from the bootstrap device.
curl -X POST https://critteria.immotus.app/api/pairing-code \
  -H "Authorization: Bearer $BOOTSTRAP_TOKEN"
# expect a JSON body with a 6-char code and expires_at (5 min out).

# Redeem it from a "second tablet" — a fresh deviceId, no auth header.
curl -X POST https://critteria.immotus.app/api/pair \
  -H 'Content-Type: application/json' \
  -d '{"code":"<the 6-char code>","deviceId":"'"$(uuidgen)"'","label":"second tablet"}'
# expect a JSON body with a device token for the new tablet.

# Confirm single-use: redeeming the same code again must fail.
curl -X POST https://critteria.immotus.app/api/pair \
  -H 'Content-Type: application/json' \
  -d '{"code":"<the same 6-char code>","deviceId":"'"$(uuidgen)"'","label":"x"}'
# expect 4xx (code already used). A 200 means the used_at guard on
# pairing_codes isn't wired — audit /api/pair before continuing, or
# any leaked code becomes reusable.
```

*Checkpoint:* `SELECT * FROM families;` in the D1 Console shows one
row. The backend is now real, and nothing in the app has changed yet.

### Step 10 — Wire the frontend (done)

`index.html` has a sync layer bolted on. A new `device` object
(`immotus-device-v1` in `localStorage`, separate from the existing
`immotus-pet-v1` pet-state key) tracks `deviceId` (generated once,
stable), `token`, `familyId`, `kidId`, and `petId`. `boot()` is the
single place that decides which of five screens to show, based on what
`device` currently has:

- No `token` + no locally-raised pet → **family-setup** screen: "First
  tablet" (enters the `SIGNUP_SECRET`, POSTs `/api/family`) or "I have a
  code" (POSTs `/api/pair`). Either way the returned token lands in
  `device` and `boot()` re-runs.
- No `token` + an existing locally-raised pet → skip straight to the
  pet screen, unchanged from the original prototype. Deliberately **not
  forced**: a device that already has a pet raised in `localStorage`
  keeps working exactly as before rather than getting interrupted by
  this feature landing. A 🔗 button next to the rename button opens
  family-setup on demand; connecting later upload-creates that pet
  server-side via `POST /api/pets` (species/colorVariant/name carried
  over, stats reset to the 80/80/80/80 default — a one-time,
  documented trade-off).
- `token` but no `kidId` → **kid-picker** screen: pick an existing kid
  (`GET /api/kids`, `POST /api/kids/:id/claim`) or add a new one (name +
  one of eight emoji avatars, `POST /api/kids` then claim).
- `token` + `kidId` but no `petId` → `resolveOrCreatePet()` calls
  `GET /api/pets`, looks for a pet already owned by this kid, and either
  adopts it or falls through to the species picker (whose click handler
  now branches: `POST /api/pets` when synced, the original local-only
  path otherwise).
- All four set → the pet screen, server-authoritative: `pollPet()` GETs
  `/api/pets/:id` every 5 s via `setInterval`, skipped when
  `document.visibilityState !== 'visible'`; Feed/Play/Clean/Sleep POST
  to `/api/pets/:id/action` and adopt the returned state
  (`applyServerPet`) instead of running the local decay/effect math.
  `localStorage` still gets written on every sync (`saveState()` inside
  `applyServerPet`), so a reload or a dropped connection falls back to
  the last-known cache rather than a blank state. A local 15 s decay
  timer only runs pre-sync; `stopLocalTick()` retires it the moment a
  server pet takes over, so the two clocks never fight over the same
  stats.

"Visit a sibling" is a new screen off a button that only appears once
synced: it lists the other kids' pets (`GET /api/pets` joined against
`GET /api/kids` for name/avatar), renders the visitor's own pet and the
selected sibling's pet side by side (`renderMiniPet` — a stripped-down
version of the main renderer: mood → pose → sprite, no color-variant
recolor, to keep it synchronous), and offers Feed/Play buttons that
POST to the sibling pet's `/api/pets/:id/action`. A 429 there (already
helped today, server-enforced per §7) surfaces as a plain status
message rather than a client-side guess at the daily limit.

Verified with a scripted Playwright walkthrough against a mocked API
(fresh device → wrong secret → correct secret → add kid → claim →
pick species → server pet created → Feed syncs → visit a sibling →
helper Feed succeeds → reload resumes without re-setup) and a second
walkthrough with every `/api/*` request forced to fail, confirming the
"Not now" link on family-setup still reaches a fully offline,
fully-playable local pet. One real bug caught by that harness before
it shipped: `setUiScreen('app')` was being called throughout instead of
`setUiScreen('app-body')` (the screen's actual element id), which
silently left every screen hidden after finishing setup — worth
knowing about since it's the kind of typo that a manual click-through
would probably have also caught, but the scripted version caught it
immediately and made the fix easy to verify.

**"Add another tablet"** followed as a fast-follow, closing the one gap
the first pass left: minting a pairing code originally required a raw
curl/PowerShell call to `POST /api/pairing-code` (the manual test in
Step 9 above), because the frontend only had the *redeem* side of
pairing (the "I have a code" form). A 📲 button next to the rename/
connect icons — visible once `device.token` exists, same gate as the
kid-picker/pet flow, since the endpoint has no "must have claimed a
kid" requirement — opens a screen that mints a code on entry, displays
it as two space-grouped triplets with a live countdown to its 5-minute
expiry, and offers "Get a new code" to remint on demand. No PowerShell
needed to add the second-through-fourth kid's tablet once the family
itself exists.

Not wired into the frontend by this pass, left for later: gifting
(`POST /api/pets/:id/gift` exists server-side, no UI button for it) and
the presence badge (`GET /api/pets/:id/events` returns one, nothing
reads it yet). Both are additive — neither touches the sync plumbing
above.

### Rollback

Every step is reversible and none of it changes the frontend PWA. If
the hosting change at Step 2 goes badly, revert the Worker deployment
and the app keeps working from `localStorage` — no data lives on the
server until Step 9. Delete the Worker and D1 database and you are
exactly where you started.

### Free-tier headroom

For a family of four playing an hour a day at 5-second polling: ~4 kids
× 720 polls/hr × 1 hr = ~2 900 reads/day, plus a handful of write
actions per kid per session. Well under CF Workers' 100 k requests/day
and D1's 5 GB free-tier limits, with plenty of room for sibling-visit
traffic on top. Star-homeschool has been running the same architecture
long enough to confirm free is realistic, not a technicality.
