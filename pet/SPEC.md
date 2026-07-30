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
  pegasus, hippocampus, phoenix, and griffin (`pet/assets/`); slime/blob is
  still placeholder SVG art — see Art pipeline below. Species is chosen
  once at first launch and then locked for that pet — see Species selection
  below.
- Mood- and action-driven **pose swapping**: the renderer picks a distinct
  drawn sprite per state (idle/happy/sad/sleep + eat/play/bath) from the
  `SPECIES_POSES` manifest, falling back to `idle` for any pose not yet
  drawn — see Art pipeline below. Only `idle` exists today for the six
  raster species, so the fallback is currently doing all the work.
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

- **64×64 canvas**, creature filling roughly 75–85% of it, leaving room for
  tails/wings/ears/effects. Never crop body parts.
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

The six stills currently in `pet/assets/` predate these rules — they're
471–640px, anti-aliased, gradient-shaded, and inconsistent in camera angle
(the T-Rex is near-profile, the griffin painterly ¾, the unicorn full side).
They are proof-of-concept placeholders, not conforming assets. Some also came
back from the generator with an opaque or faux-checkerboard background
instead of real alpha, and were cleaned up with a flood-fill background
removal pass before being cropped and added.

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

Item sprites are **32×32, on the shared palette, and species-independent** —
one apple/food item, one soap-and-bubbles, one ball, reused across all six
species. Three sprites total, no per-species multiplication.

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
so conforming 64×64 art upscales crisply while the legacy 640px stills keep
smooth downscaling. That detection is transitional and can be dropped once
every asset is 64×64.

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
