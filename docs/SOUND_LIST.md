# Helmsong — Sound Effects List (for Suno generation)

> Production notes: **one-shots** 0.5–3s, **loops** 15–30s and seamless.
> mp3 is fine (same pipeline as the music). Keep loudness consistent across the
> set — everything plays under a shared SFX volume slider. For sounds that fire
> often (cannon, hits, catches, clicks) generate **2–3 variations** each so they
> don't fatigue. Kebab-case filenames, e.g. `sfx-cannon-1.mp3`.

## Already sounded (procedural placeholder blips — replace with real files)

| # | Sound | Trigger | Type |
|---|-------|---------|------|
| 1 | Cannon fire | your shots + nearby ship volleys (5 call sites) | one-shot ×3 |
| 2 | Impact / hull hit | cannonball strikes, rams, monster bites (6 sites) | one-shot ×3 |
| 3 | Fish caught | net success | one-shot ×2 |
| 4 | Coin chime | buy/sell/pay/pickup (16 sites — most-heard sound in the game) | one-shot ×2 |
| 5 | Fanfare / success | upgrades, quest steps, boss kills, christening, pacts (12 sites) | one-shot |
| 6 | Thunder | storm lightning | one-shot ×2 |

## Ambient loops (currently procedural noise — replaceable)

| # | Sound | Trigger | Type |
|---|-------|---------|------|
| 7 | Sea wash / waves | always (swells with roughness) | loop |
| 8 | Wind | always (tracks wind strength) | loop |
| 9 | Combat tension drone | enemies near | loop (may retire once combat music carries this) |

## MISSING — Tier 1 (heard constantly, biggest impact)

| # | Sound | Trigger | Type |
|---|-------|---------|------|
| 10 | UI click | every button/tab press (`SFX.click` is referenced but doesn't exist yet) | one-shot, subtle |
| 11 | Sail raise | W — canvas snap + rope | one-shot |
| 12 | Sail reef/lower | S — canvas ruffle down | one-shot |
| 13 | Anchor drop | Shift — chain rattle + splash | one-shot |
| 14 | Net cast | Space — whoosh + splash | one-shot |
| 15 | Net haul/stow | stowing the net — wet rope haul | one-shot |
| 16 | Dock at port | harbour bell + wood bump | one-shot |
| 17 | Set sail from port | rope cast-off + creak | one-shot |
| 18 | Port ambience | while docked — gulls, pier creak, distant chatter | loop |
| 19 | Rain | rainy weather | loop (light + storm-heavy variants) |
| 20 | Ship collision | hitting land/bergs/ships — heavy wood thud | one-shot ×2 |
| 21 | Reef scrape | crossing shoals too fast — grinding hull | one-shot/short loop |
| 22 | Ship explosion/sink | enemy ship destroyed — burst + groan | one-shot ×2 |
| 23 | Your ship sinking | death — long creak + water rush | one-shot (3s) |
| 24 | Cannonball splash | missed shots hitting water | one-shot ×2 |

## MISSING — Tier 2 (character & feedback)

| # | Sound | Trigger | Type |
|---|-------|---------|------|
| 25 | Leviathan rise + roar | boss spawn — deep sea-monster bellow | one-shot (3–4s) |
| 26 | Monster roars ×5 | region terrors (kraken, wyrm, jelly zap, deepmaw, angler bite) | one-shots |
| 27 | Monster death | any beast slain — gurgling collapse | one-shot |
| 28 | Tentacle strike | leviathan/kraken slam — big splash | one-shot |
| 29 | Fire ignition + burning | fire shot, burning slicks/ships | one-shot + short loop |
| 30 | Chain shot | whirling chain fire variant | one-shot |
| 31 | Grapple hook | bite into hull + rope strain | one-shot |
| 32 | Fort cannon | shore-gun boom (deeper than ship cannon) | one-shot |
| 33 | Fort falls | siege win — stone collapse rumble | one-shot |
| 34 | WANTED sting | committing piracy — alarm-ish sting | one-shot |
| 35 | Low hull alarm | hp under ~25% — creaking warning | one-shot, throttled |
| 36 | Lantern light / douse | L key — match strike / puff | one-shot ×2 |
| 37 | Flotsam/loot pickup | wet grab / crate clunk (distinct from coin) | one-shot |
| 38 | Whirlpool churn | near whirlpools | loop |
| 39 | Iceberg groan/crack | Frostreach near bergs | one-shot ×2 |
| 40 | Death sting | "THE SHIP WAS LOST" screen | musical sting (3s) |
| 41 | Victory sting | boss/terror slain | musical sting (3s) |

## MISSING — Tier 3 (flavor & wildlife)

| # | Sound | Trigger | Type |
|---|-------|---------|------|
| 42 | Gull cries | fishing grounds, near islands | one-shot ×3, sparse |
| 43 | Whale song + spout | whale sightings | one-shot |
| 44 | Dolphin chirp + leap splash | dolphin pods | one-shot |
| 45 | Sea-sprite shimmer | sprite sightings — soft magical glint | one-shot |
| 46 | Map open/close | M — paper/parchment rustle | one-shot ×2 |
| 47 | Captain's screens | I — ledger/book flip | one-shot |
| 48 | Ghost ship moan | ghost ships near — faint wail | one-shot, sparse |
| 49 | War zone distant battle | contested waters — far cannon rumble | loop |
| 50 | Stronghold claimed | raising your banner — horn + flag snap | one-shot |
| 51 | Quest sting (Deep Song) | quest chain complete — grander fanfare | musical sting (4–5s) |

**Suggested first batch (12 files):** 10 click · 11/12 sails · 13 anchor · 14 net cast ·
16 dock bell · 18 port loop · 19 rain loop · 20 collision · 22 ship sink ·
24 splash · 25 leviathan roar · 1 cannon (real).
