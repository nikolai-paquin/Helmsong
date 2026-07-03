# Helmsong — Expansion Plan (V9 → V14)

> The second era of Helmsong: a living, contested sea. This plan groups the full
> feature wishlist into six dependency-ordered phases. Each phase is shippable and
> verifiable on its own (per house rules: verify in preview, console clean, keep
> the lofi/chill tone — danger is something you *seek out*, not something that
> hunts you by default).
>
> Version numbers continue from V8 (the designed world). Every wishlist item is
> mapped to a phase — see the traceability table at the bottom.

---

## Guiding principles

1. **Foundation before excitement.** Forts need real islands. Factions need forts
   and NPC-vs-NPC combat. The player's empire needs factions to ally with or
   fight. Build the layers in that order.
2. **The sea stays chill by default.** Faction wars are *localized* — contested
   waters are marked on the chart and colored on the sea; peaceful trade lanes
   stay peaceful. A new player sailing the Calm Shallows should feel exactly the
   game we have today.
3. **Everything visible, nothing abstract.** Storms are objects you see coming.
   Loot floats where the ship sank. Territory is flags on forts, not a menu stat.
4. **One file, code-drawn.** All of this stays procedural canvas art in
   `index.html`, in the established VESSELS/chunk/region patterns.

---

## Phase V9 — The Living Sea *(world & environment foundation)* — ✅ SHIPPED
> Batches 1–4 all live (see HANDOFF §6): organic terraced islands + rocks/shelves,
> reef/shoal/sandbar scraping, physical roaming squalls, icebergs/riptides/whirlpools.
> Open nice-to-haves: chart draws island polygons; NPCs respect hazards.

**Goal:** the ocean itself becomes the first "faction" — terrain and weather you
read, dodge, and route around. Everything later sits on this.

- **Island rework — bigger, organic shapes.** Reference images received (low-poly
  isometric islands + a ragged pixel-art landmass map). Extracted art direction:
  1) **organic ragged coastlines** — bays, headlands, peninsulas (strong low-freq
  lobes + fbm detail on a radial polygon); 2) **pale shallow-water shelf** ringing
  every island (signature look; later doubles as the shoal/damage zone);
  3) **satellite islets & rocks** scattered around bigger islands (sea stacks,
  lighthouse-outcrop vibes) — gives the archipelago feel without breaking
  star-convex collision; 4) **terraced elevation** — beach ring → grass tiers →
  rocky peak (stacked shrunken coast polygons with cliff walls); 5) landmark
  silhouettes (spire rocks, hilltop clusters). Collision stays per-angle radius
  (`islandRadiusAt` pattern). Sizes up ~2×, density down to compensate.
- **Reefs, shoals, sandbars.** Sub-surface hazard rings around islands + free-
  standing patches: visible as water-color/foam texture, damage + slow on
  contact (scaled by speed). Spyglass/chart reveals them; local knowledge matters.
- **Storms as physical, moving weather cells.** Kill the full-screen weather
  overlay: `storms[]` = world-anchored cells `{x, y, r, vx, vy, intensity}` that
  drift across the map. Inside one you get today's storm effects (rain, hull
  stress, gusts); outside you can *watch it pass*. Visible on the horizon
  (darkened sea + rain column), on the minimap, and on the chart — so you can
  outrun or outmaneuver it. Forecast panel becomes "storm bearing/distance".
  `weather.state` becomes a *derived, local* value (`stormAt(x,y)`).
- **More water hazards:** icebergs (Frostreach; physical colliders that drift),
  riptides (current bands that shove the ship, shown as streaked water), and
  whirlpools (pull + damage at the core, escape by sailing across the spiral).
- Debug: `__HS.storms`, `spawnStorm(x,y)`, `spawnHazard(type)`.

**Why first:** changes the *sailing* core loop (pillar #1), and every later system
(forts on islands, faction territory, empire) assumes this terrain exists.

---

## Phase V10 — Gunnery & Spoils *(combat depth + inventory)* — ✅ SHIPPED
> Batches 1–3 all live (see HANDOFF §6): floating loot drops + boat naming; Ship's Hold
> inventory (cargo grid, price ledger, ammo rack, I key / port tab); ammo behaviours
> (chain/grapple/grape/fire), utility shots (oil slick + chum), crates-of-5 ammo shop,
> enemy ammo flavour by tier. Open nice-to-haves: enemies using grapple/utility shots;
> NPC ships respecting slicks (shared with the V9 NPC-hazard-parity follow-up).

**Goal:** fights have texture — you choose what to fire, and victory physically
litters the sea with loot.

- **Ammo types (combat):** round shot (default), **chain shot** (shreds sails →
  slows target), **grapple shot** (pulls you together for point-blank exchanges,
  sets up boarding later), **shotgun/grape blast** (short-range cone, multi-hit),
  **fire shot** (ignites ships — burning DoT that can spread to sails).
- **Utility shots:** **oil slick** (floats on water, ignitable by fire shot —
  area denial), **chum shot** (attracts fish to a spot… and sharks — dual use:
  fishing buff or monster bait).
- **Ammo economy:** shots other than round shot are crafted/bought at ports and
  consumed. Enemies get ammo types by tier/faction (navy uses chain, pirates use
  fire, etc.) so fights read differently per opponent.
- **Loot drops in the world.** No more auto-loot: kills spill floating crates/
  barrels/coin at the sink point (bobbing, despawn after ~90s, magnet-pickup on
  sail-over). Risk/reward: grabbing loot mid-fight is a choice.
- **Inventory screen (I key / port tab):** cargo grid with icons + counts,
  **ammo slots** (select active shot; also selectable from the action bar),
  and a **price ledger** — the game records prices *you have seen* at visited
  ports, so the inventory shows "Rum: paying 46c at Dunwick (seen 2 days ago)".
  Knowledge-as-progression, fits the logbook fiction.
- **Name your boat** (small QoL, fits here with the identity theme): set at the
  shipyard / first launch; shown in the port header, logbook, and death screen.

**Depends on:** nothing from V9 strictly (parallelizable), but loot-in-water is
nicer after V9's swim-able hazard water exists.

---

## Phase V11 — Trade Winds *(economy depth)* — ✅ SHIPPED
> All four bullets live (see HANDOFF §6): 8 region-exclusive goods + 3 endemic fish with
> origin-distance pricing (climate-band distance sets the premium; exotics stocked only
> at source ports); fish-school depletion (player + NPC fishers, visible thinning,
> ~2-day regen); ledger v2 (per-good price book, best market scored by price × wrapped
> distance, dock rumours leak far-off prices).

**Goal:** geography = economy. Traveling far pays, and the world's resources are
finite enough to feel alive.

- **Region-specific goods & fish.** Each climate/culture region gets exclusive
  produce (silk, jade, amber, obsidian, papyrus, olive oil…) and endemic fish
  species. Ports pay big for goods far from their origin → real trade routes
  worth planning on the chart.
- **Fish school depletion.** Fishing grounds carry stock that depletes as you
  *and NPC fishing boats* work them (visible: fewer birds/ripples as a ground
  thins), regenerating over days. Chum (V10) temporarily concentrates fish.
- **Price ledger v2:** the inventory's "where to sell" view sorts by best known
  price × distance; rumors at ports can leak far-off prices.
- **Market simulation touch-up:** origin-distance pricing replaces some of the
  flat produce/demand tables; keeps the existing event system (gluts/shortages).

**Depends on:** V10's inventory/ledger UI.

---

## Phase V12 — Banners of the Sea *(factions & the living world)* — ✅ SHIPPED
> All core systems live (see HANDOFF §6): 5 factions (Crown/Guild/Shorefolk/Red Sails/
> Tidebound) with port ownership, per-faction reputation with price + hostility
> consequences, faction-aware NPC-vs-NPC combat (pirates raid merchants, navy hunts
> pirates, monsters bite anyone), rare localized war zones with visible skirmishes,
> and attackable garrisoned forts incl. the hidden Tidebound stronghold (destroy for
> plunder; forts rebuild in ~3 days). Deferred: relations drift over time; faction-vs-
> faction fort captures (folded into V14's capture mechanics); distinct guild warship.

**Goal:** the sea has owners. This is the biggest single system.

- **Faction system core.** `FACTIONS{}`: identity (name, colors, flag, culture),
  home territory (regions/islands owned), fleet composition (which VESSELS they
  field), and a **relations matrix** (war/hostile/neutral/friendly/allied) that
  drifts over time and with events.
- **Territory on the map.** Islands and forts have owners; the chart tints
  contested/owned waters; guide labels show whose waters you're entering.
  Reputation per faction: attack their ships → their patrols hunt *you*
  (generalizes the existing navy notoriety/bounty system).
- **NPC-vs-NPC combat.** Ships and monsters pick targets by faction hostility,
  not just the player: pirate raids on merchants, navy hunting pirates, rival
  faction skirmishes in contested water, monsters attacking anyone. You can
  watch, join, or profit (loot drops from V10 apply to *any* kill).
- **Strongholds & forts.** Island structures with garrisons: pirate hangouts,
  navy bases, trading-guild houses, fishing-guild docks. Visible, attackable
  (shore cannons fire back, garrison sorties), and rewarding: destroy for loot,
  or (V14) capture. Forts are the physical anchor of territory.
- **Minor factions & cults (v1).** 2–3 small cults with a single hidden
  stronghold, unique ship skins and a signature monster (the angler/kraken/
  leviathan roster gets faction affiliations — "the Cult of the Deep Song").
- **Launch cultures:** ship with 3–4 distinct factions (e.g., the Navy/European
  crown, a pirate confederacy, a trading guild, one cult) to prove the system
  before the full culture expansion.
- Debug: `__HS.FACTIONS`, `rep()`, `setWar(a,b)`, `spawnFort(type)`.

**Depends on:** V9 (islands to hold forts), V10 (loot, ammo variety for faction
flavor), V11 (guild economies mean something).

---

## Phase V13 — Many Flags *(culture theming expansion)* — ✅ FIRST SLICE SHIPPED
> The culture SYSTEM + 3 full kits are live (see HANDOFF §6): `cultureAt` maps climate
> bands + hemisphere to kits; Norse (longships, stave halls, -vik names, furs/timber),
> Dawnland/East Asian (junks, pagodas, -shima names, tea/silk), Sunborn/Desert (dhows,
> obelisks, El- names, spice/salt), with the Midlands as the existing baseline. Port
> names, captains, market produce, shore architecture, pier pennants, and civilian
> hulls all follow culture. Remaining kits (Viking≈shipped as Norse; Mayan, African,
> Greek, Egyptian, Roman, Japanese-distinct) are per-kit content drops on this
> scaffold; per-culture cults and music flavour also deferred.

**Goal:** each part of the world looks, sails, and builds differently.

- **Cultural kits**, applied per region + faction: ship silhouettes (VESSELS
  variants), building/port architecture, flags/colors, port names, market
  produce, crew names, music flavor. Planned kits:
  - **East Asian** (junks, pagoda ports), **Japanese** (distinct from the
    former — castles, red gates), **Viking/Scandinavian** (longships, stave
    halls), **European** (the existing baseline — square-riggers, stone forts),
    **Mesoamerican/Mayan** (stepped pyramids, canoe-catamarans), **African**
    (dhows, adobe citadels), **Greek** (triremes, columned harbors),
    **Egyptian** (reed barques, obelisks), **Roman** (galleys, legion forts).
  - Placement maps kits onto the existing climate bands + continents so
    cultures feel geographically coherent (e.g., Vikings toward Frostreach).
- **More cults & uniques:** each culture gets 1 associated cult/deity with a
  unique monster or unit (leans on the V12 cult scaffold).
- This phase is mostly *content* on V12's *system* — it can ship culture-by-
  culture across several sessions, each independently verifiable.

**Depends on:** V12 (factions are what get themed).

---

## Phase V14 — Your Flag *(player empire — capstone)* — ✅ SHIPPED
> The capstone slice is live (see HANDOFF §6): fleet escorts (buy, formation follow,
> auto-gunnery, screening, loss & repair), claimable stronghold from any razed fort
> (defensive guns, warehouse vault, your territory on the chart and in the waters),
> and founding your faction (name + banner colour; escorts/fort/chart follow it;
> powers react by standing) with two diplomacy levers (Red Sails tribute truce, Guild
> trade-pact). Deferred to the post-plan backlog: automated fleet trade routes,
> sieges against the player's hold / fort captures, soft win-condition flavours.
>
> **⚓ THE V9→V14 EXPANSION PLAN IS COMPLETE.** Every phase shipped in verified
> batches. The living backlog now lives in HANDOFF §6's deferred notes.

**Goal:** the player graduates from a captain to a power on the map.

- **Fleet building.** Buy/capture additional ships; they sail with you as
  escorts (formation follow, focus-fire orders) or run **automated routes**
  (trade/fish a circuit, generating passive income — attackable by pirates, so
  routes need protecting). Fleet screen in the inventory/port UI. Each ship
  keeps its own name (V10 naming), class, and upgrades.
- **Your stronghold.** Buy/repair a ruined fort (or capture one in war):
  upgrade its defenses, warehouse (remote cargo storage), shipwright, and
  recruiting hall. It projects a control radius = your first territory.
- **Found your faction.** Name it, pick a flag/colors; existing factions react
  (guilds court you, crowns tax you, pirates test you).
- **Diplomacy & war.** Ally, trade-pact, or declare war on factions; wars are
  fought over forts (siege/defend events), and territory changes hands. Win
  conditions are soft: trading empire (route income), pirate empire (tribute/
  fear), fishing guild (ground monopolies) — the sandbox fantasy, not a
  score screen.

**Depends on:** everything — V12 factions/forts, V10 fleet-able combat + loot,
V11 route economics, V9 terrain worth owning.

---

## Quick wins (small, can slot into any session)

| Item | Size | Natural home |
|---|---|---|
| Name your boat | XS | V10 (or anytime) |
| Loot drops → sail-over pickup | S | V10 |
| Fish school depletion | S | V11 |
| Inventory v1 (cargo + ammo view) | M | V10 |

## Traceability — every wishlist item → phase

| Wishlist item | Phase |
|---|---|
| Factions own map areas, at war | V12 |
| NPCs/monsters/factions attack each other | V12 |
| Storms as moving world areas you can outrun | V9 |
| Bigger, natural-shaped islands (low-poly refs) | V9 *(need refs re-shared)* |
| Strongholds/forts on islands, attackable | V12 |
| Reefs, shoals, sandbars | V9 |
| Build fleet, stronghold, territory, empire | V14 |
| Your faction: war/ally with others | V14 |
| Asian-themed region (boats, buildings, factions) | V13 |
| Viking, European, Japanese, Mayan, African, Greek, Egyptian, Roman kits | V13 |
| Small unique cult factions w/ unique ships & monsters | V12 (v1) + V13 (expanded) |
| Icebergs, riptides, whirlpools | V9 |
| Fish schools deplete (player + NPC) | V11 |
| Unlockable shot types (chain, grapple, shotgun, fire) | V10 |
| Utility shots (oil slick, chum) | V10 |
| Region-specific fish/materials/trade | V11 |
| Loot drops where enemies die, pick up by sailing over | V10 |
| Inventory: cargo, ammo slots, trade prices & where to sell | V10 (+V11 ledger v2) |
| Name your own boat | V10 |

## Decisions (answered by the user)

1. **V9 island refs:** ✅ received (low-poly iso islands + ragged pixel map);
   art direction extracted above. Size answer implicit: ~2× shipped in Batch 1.
2. **Faction war heat:** ✅ **rare and localized, marked as contested water on
   the chart** — the default sea stays chill; peaceful players can route around.
3. **Forts CAN fall:** ✅ forts can be **damaged, blockaded, or captured by
   other factions** (player forts included). Capture is a setback, not a game
   over — a captured fort can be retaken (keeps the "death is a setback" pillar).

## Open questions

4. **Save format:** V12/V14 add a lot of world state (faction relations, fort
   ownership, fleet). Plan a save-version bump + migration (or accept resets
   per major phase, as with V8).

## Suggested order of work

**V9 → V10 → V11 → V12 → V13 → V14**, with quick wins sprinkled in. Each phase
lands as several verified batches (the project's established cadence). V13 can
interleave with V14 (culture kits are independent content drops).
