# Immotus Virtual Pet — Spec

A Tamagotchi-style virtual pet app for four siblings, built as a web app/PWA
(Silk browser on Fire tablets), backed by Cloudflare.

## 1. Current prototype

`pet/index.html` is a working local-only prototype:
- Stats (hunger, happiness, energy, cleanliness) decay based on real elapsed
  time (computed from a stored timestamp), so the pet keeps "aging" even
  while the tablet is closed.
- Actions: Feed, Play, Clean, Sleep.
- Species picker: real AI-generated pixel-art stills for T-Rex, unicorn,
  pegasus, hippocampus, phoenix, griffin, and dragon (`pet/assets/`); slime/blob is
  still placeholder SVG art — see Art pipeline below. Species is chosen
  once at first launch and then locked for that pet — see Species selection
  below. T-Rex, unicorn, dragon, hippocampus, and pegasus are the kept
  roster (all five conform to the Art pipeline rules below); phoenix and
  griffin are legacy placeholders slated for removal, not yet acted on.
- Mood- and action-driven **pose swapping**: the renderer picks a distinct
  drawn sprite per state (idle/happy/sad/sleep + eat/play/bath) from the
  `SPECIES_POSES` manifest, falling back to `idle` for any pose not yet
  drawn — see Art pipeline below. The five kept species each have all four
  required poses (idle/happy/sad/sleep); action poses (eat/play/bath) are
  still unbuilt for all of them, so the fallback chain does that work.
- `localStorage` persistence.

Everything below is the target end-state, to be built in phases on top of
this prototype.

## 2. Architecture

```
[Fire tablet Silk browser / PWA]
        |  fetch + WebSocket
        v
[Cloudflare Pages]  --serves-->  static PWA (HTML/CSS/JS, service worker)
        |
        v
[Cloudflare Workers]  --API layer-->
        |-- D1 (SQLite)      → family, kids, pets, items, locations, visit log
        |-- Durable Objects  → one per pet: authoritative live state + WebSocket fanout for visits
        |-- KV (optional)    → session tokens / family passcode lookup
        |-- Web Push (opt.)  → "your pet is hungry" notifications
```

Each pet's Durable Object is the single authoritative owner of that pet's
live state (stats, equipped items, location, stage) and the thing that
broadcasts changes to anyone currently visiting. D1 holds everything that
doesn't need real-time locking.

## 3. Data model (D1)

- `families` — id, passcode_hash, name
- `kids` — id, family_id, display_name, avatar, pin
- `pets` — id, kid_id, species, color_variant, stage, created_at,
  total_care_points, current_location_id, durable_object_id
- `pet_equipped_items` — pet_id, slot, item_id
- `items` (catalog) — id, slot (head/neck/back/…), name, image_asset,
  species_compatibility, unlock_type
- `locations` (catalog) — id, name, image_asset, unlock_type
- `pet_events` — id, pet_id, actor_kid_id, action, timestamp (append-only
  log; doubles as audit trail and visit history)
- `visits` — id, visitor_kid_id, host_pet_id, started_at, ended_at

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
cells 1–2 of a generated 8-pose sheet, via the cleanup below.

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
written up.

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
parallel. Both are keepers — the roster is **T-Rex, unicorn, dragon,
hippocampus, and pegasus**, all five now at the required four-pose tier.
`griffin` and `phoenix` are the ones being dropped: their stills predate
these rules — 471–640px, anti-aliased, gradient-shaded, and inconsistent in
camera angle (the griffin painterly ¾, formerly the unicorn full side) —
and won't be re-cut. That removal (species picker, `SPECIES_POSES`, and the
asset files themselves) hasn't been done yet.

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
pose. Eight poses × six species = 48 creature sprites, plus 3 shared item
sprites (below); that is the real art budget for this feature, and it is the
reason for the tiering.

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

Item sprites are **64×64, on the shared palette, and species-independent** —
one apple/food item, one soap-and-bubbles, one ball, reused across all six
species. Three sprites total, no per-species multiplication. (Half the
creature canvas, scaled with it when the canvas rule changed.)

Items must be **separate files**, never drawn into a pose. The first
generated sheet baked food into four of its eight cells, which breaks the
approach phase entirely — there is nothing left to fly in.

Where an item lands is per-species, since a hippocampus's mouth is nowhere
near a T-Rex's: extend the accessory anchor config with an `item` slot, so
the food arrives at the mouth and the suds land over the body. Suds may also
sit as a persistent overlay for the reaction phase rather than vanishing on
contact.

Deliberately out of scope for now: Sleep gets no travelling item (nothing is
being applied to the pet — the pet changes state), and no action gets a
screen-shake, flash, or particle burst. The tone in §1 is calm and cozy;
these should read as gentle, not as combat feedback.

### Fallback chain

`SPECIES_POSES` in `pet/index.html` lists only files that exist. Anything
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

Attach `pet/assets/reference/trex-reference-4x.png`, not the 128×128 asset —
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
   `pet/assets/<species>-<pose>.png`. (`idle` keeps the bare
   `pet/assets/<species>.png` name it shipped with.)
2. Add the one-line entry to `SPECIES_POSES` in `pet/index.html`. The
   manifest is explicit on purpose — deriving paths by convention would 404
   into a broken-image icon for every pose not yet drawn.
3. Check it with `pet/index.html?pose=<pose>`, which pins that pose and its
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
decay — no cron needed). v1: time-gated only. Later enhancement: also
require a minimum average care score to advance, so pure neglect doesn't
still age the pet up — deliberately deferred since it adds complexity kids
may find punishing/confusing at first.

### Species selection
Species is picked once, at first launch, and then locked for the life of
that pet — `pet/index.html` shows a dedicated "Choose your pet!" screen
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
`pets.current_location_id` swaps the background image behind the pet.
Cheapest feature here — no per-species art, no anchor math, just an image
swap. Unlockable the same way as accessories, or free to switch among
whatever's unlocked.

### Color/variation at species selection
At creation, alongside picking species, the kid picks a color variant
(stored as a named preset).

Originally specced as a CSS `hue-rotate` filter over the base sprite. That
no longer holds once §4 locks a shared palette: hue-rotating by an arbitrary
angle produces colors that are by definition off-palette, and it discolors
incidental parts (teeth, eye whites, mouth interior) along with the intended
body color — visible on the current unicorn, whose gold hooves and horn shift
with its mane.

Instead, each species ships **2–3 hand-picked palette swaps**: a small map of
`{ source palette index → replacement palette index }` per variant, applied
once per sprite at load time on an offscreen canvas and cached. Bounded work
(2–3 short mappings per species, not an image per color × species), stays on
palette, and leaves the parts that shouldn't change alone. Because it's a
palette remap rather than a filter, it applies uniformly across every pose
without per-pose tuning.

### Simultaneous own + sibling pet during visits
The one item here with real backend weight. Visiting a sibling means
rendering two pets side-by-side: the visitor's own pet (local/cached) and
the host's pet fetched from their Durable Object. Two implementation
options:
- **v1 (recommended to start): polling.** Fetch the host pet's current
  state/location/accessories every few seconds while visiting. Simple to
  build, no WebSocket infrastructure needed, "live enough" for kids taking
  turns caring for each other's pets.
- **v2: WebSocket push.** The host's Durable Object holds open WebSocket
  connections and broadcasts state changes instantly to any connected
  visitor — needed if we want truly simultaneous co-visit (both kids acting
  on the host pet at once, or a synchronous two-pet mini-game later).

Recommendation: build polling first: it satisfies "see both pets on screen
at once" fully, and the Durable Object model means upgrading to WebSocket
push later doesn't require a data-model change, just a transport change.

## 6. Social / visiting features (recap)

Since all four kids are one family, "friends" is just the sibling roster —
gated by a family passcode, no add-friend flow. Recommended v1 slice:
read-only visiting + once-a-day helper actions (feed/play on a sibling's
pet) + gifting + presence indicator. Live co-visit and any mini-games stay
deferred until the core loop and the polling-based visit view are solid.

## 7. Identity/auth

No email/password. Kid picks their avatar from the family's kid icons, then
a 4-digit PIN (or skipped entirely if the tablet is already locked to one
Fire OS kid profile). Family-level passcode gates the Worker API.

## 8. Build phases

1. **MVP (in progress)**: single pet, stats/mood engine, species picker,
   local persistence — this is `pet/index.html` today.
2. **Real art (in progress)**: pose-swapping renderer and fallback chain are
   built. Remaining, in order: commit the shared palette; draw the four
   required poses per species (`sleep` first — it's the one whose absence is
   currently most obvious); build the travelling-item choreography and the
   shake/bounce loops (CSS, no art dependency); draw the four action poses
   including the `eat-chew` second frame and the three shared item sprites;
   then re-cut the six placeholder stills to 64×64 so the whole set conforms
   to §4.
3. **Cloudflare backend**: Workers + D1 + Durable Objects, family/kid
   profiles, passcode auth, cross-device sync.
4. **Social v1**: read-only visiting, once-daily helper actions, gifting,
   presence badges.
5. **End-state features**: lifecycles/aging, accessories, locations, color
   variants, polling-based simultaneous two-pet visit view.
6. **Nice-to-haves**: WebSocket live co-visit, push notifications,
   mini-games, guestbook/stickers, family (non-competitive) garden score.
