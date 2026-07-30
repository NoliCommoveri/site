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
  pegasus, and hippocampus (`pet/assets/`); slime/blob is still placeholder
  SVG art — see Art pipeline below.
- Mood-driven visual states (happy/neutral/sad/sleeping/dirty) via CSS, no
  hand animation required.
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

## 4. Art pipeline (no art skills required)

Because hand-coded SVG shapes don't read as "real" to young kids, the art
comes from AI-generated pixel-art stills (ChatGPT image gen), one clean
image per species with a transparent background. A single "idle/happy" pose
per species is enough; other moods (sad, sleepy, dirty) are faked with CSS
filters and overlays (desaturate + droop, dim + "Zzz" + slow bob, semi-
transparent dirt smudge) rather than needing a hand-drawn pose for every
emotion.

T-Rex, unicorn, pegasus, and hippocampus stills are in `pet/assets/`
(`trex.png`, `unicorn.png`, `pegasus.png`, `hippocampus.png`) and wired up
in `pet/index.html`: an `<img>` sprite swaps in per species alongside the
original blob SVG, with the mood/dirt/sparkle overlays now implemented as
plain HTML elements positioned over the stage so they work uniformly across
both the SVG and raster-sprite species. Some source stills came back from
the generator with an opaque or faux-checkerboard background instead of
real alpha transparency; those were cleaned up with a flood-fill background
removal pass before being cropped and added.

Rendering is layered, back to front:
1. Location/background image (behind everything)
2. Base species sprite, with a CSS `hue-rotate`/`saturate` filter applied
   for the chosen color variant (avoids needing a separately generated
   image per color)
3. Accessory overlays (hat, scarf, etc.), positioned via a small per-species
   anchor config (`{ species, stage } → { slot: {x, y, scale} }`)
4. Mood-driven CSS animation/filters on top (bounce, sparkle, droop, Zzz)

Additional sprites needed over time: one base image per species per
lifecycle stage (egg/hatchling/young/teen/adult), plus one image per
accessory item. Locations are just background images — no per-species work.

## 5. End-state feature breakdown

### Lifecycles (aging)
`pets.stage` advances egg → hatchling → young → teen → adult based on
elapsed real time since `created_at` (computed lazily, same pattern as stat
decay — no cron needed). v1: time-gated only. Later enhancement: also
require a minimum average care score to advance, so pure neglect doesn't
still age the pet up — deliberately deferred since it adds complexity kids
may find punishing/confusing at first.

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
(stored as a hue value or named preset). Applied at render time as a CSS
filter over the one base sprite — not a separate generated image per color.
Trade-off: hue-rotate can slightly shift incidental colors (e.g. teeth,
mouth interior) along with the intended body color; acceptable for a
cosmetic kids' feature, and avoids multiplying the art-generation workload
by every color × species combination.

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
2. **Real art**: swap placeholder shapes for AI-generated species stills
   (T-Rex, unicorn, more later), mood faked via CSS over the one image.
3. **Cloudflare backend**: Workers + D1 + Durable Objects, family/kid
   profiles, passcode auth, cross-device sync.
4. **Social v1**: read-only visiting, once-daily helper actions, gifting,
   presence badges.
5. **End-state features**: lifecycles/aging, accessories, locations, color
   variants, polling-based simultaneous two-pet visit view.
6. **Nice-to-haves**: WebSocket live co-visit, push notifications,
   mini-games, guestbook/stickers, family (non-competitive) garden score.
