# Helmsong — Game Design Document

> A lofi, infinite-sea sailing game. You are a sailor-merchant in an 1800s fantasy
> world: trawl for fish, fight sea monsters and pirates, find islands, and trade
> cargo between ports. Wind Waker's sailing joy meets a chill fishing/trading sim.

---

## 1. Pillars (what every decision serves)

1. **The sea feels alive.** Waves, wind, weather, light, and sound make sailing
   itself the reward — even with nothing to do, the ocean is pleasant to be on.
2. **Lofi & low-pressure.** Chill core loop. Danger exists but the default mood is
   calm. Death is a setback, not a punishment. Sessions of 5 minutes or 2 hours
   both feel good.
3. **Infinite & emergent.** A procedurally generated, endless world. Every voyage
   is a little different. Islands, monsters, weather, and markets recombine.
4. **Honest systems.** Wind direction matters. Cargo weight matters. Storms are
   real threats. The simulation is simple but consistent, so mastery is possible.

## 2. The Fantasy

You inherit a small fishing sloop and a logbook. The world is an endless archipelago.
You make your living however you like: a careful trawler who never leaves calm water,
a monster-hunter chasing bounties, a smuggler running cargo through storms for the
best margins, or an explorer who just wants to see what's past the next island.

Tone: weathered wood, brass instruments, oil lamps, sea shanties hummed under breath.
Fantasy is *light* — leviathans and a touch of the uncanny, not high magic.

---

## 3. Core Loop

```
        ┌─────────────────────────────────────────────┐
        │                                             │
   SAIL ──▶ DISCOVER ──▶ ACT (fish / fight / trade) ──┘
   (wind,    (islands,    │
    waves,    monsters,   ▼
    weather)  ports)   EARN coin + cargo + story
                          │
                          ▼
                    UPGRADE ship & gear ──▶ go further / survive worse
```

- **Sail:** read the wind, manage the sail, ride or avoid waves and storms.
- **Discover:** the map reveals as you go. Islands, wrecks, monster pods, ports.
- **Act:** the three activity loops (fish, fight, trade) overlap on the same sea.
- **Earn → Upgrade:** coin and materials buy a bigger hull, better nets, cannons,
  a stronger sail — which open rougher waters and richer rewards.

---

## 4. Systems

### 4.1 Sailing & Wind (the foundation — V1)
- Top-down (¾ camera tilt optional later). Ship has heading, throttle (sail open
  amount), and momentum. Turning is gradual; the sea has drift.
- **Wind** has a direction and strength that drifts over time. Sailing with the
  wind is fast; against it is a slow tack. A wind indicator (compass rose) shows it.
- **Sail control:** raise/lower/trim the sail. Full sail downwind = speed; you must
  reef the sail in storms or risk damage.
- **Waves** push the boat, rock the sprite, and affect handling. Bigger seas in
  rougher regions.

### 4.2 World Generation
- Infinite ocean via chunked procedural generation (deterministic from a world seed
  + chunk coords). Value/Perlin-ish noise drives:
  - **Landmass** (islands, their size/shape/biome)
  - **Depth** (shallows, open sea, deep trenches — affects fish & monsters)
  - **Region temperament** (calm / choppy / storm-prone / cursed waters)
- Points of interest seeded per-chunk: ports, fishing grounds, monster pods,
  wrecks/treasure, pirate lanes.
- Map memory: discovered chunks persist; a logbook/minimap fills in.

### 4.3 Weather & Time
- Day/night cycle (lighting, color grade, visibility).
- Weather states: **clear → cloudy → rain → squall → storm**, transitioning over
  time and by region. Storms = big waves, wind gusts, lightning, low visibility,
  hull stress. Avoiding/surviving storms is core tension.
- Fog banks reduce sight; calm doldrums kill your wind.

### 4.4 Fishing / Trawling
- Drop a net or line over **fishing grounds** (revealed by bird flocks, ripples).
- **Trawling:** lower the net, sail slowly, drag for a time; net fills with a catch
  table weighted by region/depth/time/weather. Risk: snag, or net too full = drag.
- **Line/rod:** active minigame for bigger/rarer fish (tension bar).
- Fish → sell at port, or use for crew morale / bait / crafting.

### 4.5 Combat
- **Sea monsters:** pods/leviathans with telegraphed attacks (tentacle slams,
  charges, whirlpools). Fight with cannons / harpoons; positioning + wind matter.
- **Pirate ships:** broadside cannon duels; board for loot, or outrun.
- Player ship has hull HP, can take on water (slows you, must bail/repair).
- Loot: coin, rare materials, monster parts (high-value trade goods).

### 4.6 Trading & Economy
- **Ports** on islands have a market: buy/sell goods with fluctuating prices driven
  by supply/demand + region (a fishing village pays well for tools, cheap for fish).
- **Cargo hold:** limited slots/weight. Weight affects speed & handling.
- Arbitrage: buy low here, sell high there. Prices drift; events (storm cut off a
  port, monster attack) spike demand. Contracts/bounties for directed goals.

### 4.7 Ship & Progression
- Upgrade: **hull** (HP, cargo), **sail** (speed, storm tolerance), **net/gear**
  (catch quality/quantity), **cannons/harpoon** (combat), **instruments** (map
  range, weather forecast).
- Soft progression: better ship → reach rougher/richer regions → bigger rewards.
- No hard level gates; the sea itself gates you (can your hull survive out there?).

---

## 5. Controls (V1 default — keyboard)

| Input | Action |
|---|---|
| `W` / `S` | Sail open / reef (throttle) |
| `A` / `D` | Turn helm left / right |
| `Shift` | Drop anchor / brake |
| `Space` | Context action (cast net, fire, interact at port) |
| `M` | Map / logbook |
| `Esc` | Pause |

Gamepad + touch later. The boat must feel good with just A/D/W/S.

---

## 6. Art & Audio Direction

- **Procedural canvas art.** Everything drawn in code: layered animated water
  (gradient + scrolling wave bands + foam), the ship (hull/sail/wake), islands
  (beach→grass→trees), monsters, weather particles (rain, spray, lightning).
- **Lofi palette:** muted, warm, low-saturation. Soft sea blues/teals, sandy warms,
  storm grays. Color grade shifts with time-of-day and weather.
- **Camera:** follows ship, slight lookahead in travel direction, subtle sway.
- **Audio (later):** ambient sea + wind bed, lofi music, dynamic intensity in storms
  & combat. Web Audio, procedural where possible. (Stub in V1.)
- Reference vibe: the inspiration boats — top-down pixel sailboats, calm seas,
  Wind Waker's "great sea."

---

## 7. Technical Architecture

- **Single `index.html`**, HTML5 Canvas 2D, vanilla JS. No build step. (Matches the
  established portfolio-game workflow: serve a copy from `/tmp`.)
- **Game loop:** fixed-timestep update + interpolated render; `requestAnimationFrame`.
- **Modules (in-file sections):**
  - `Input` — key state.
  - `World` — seeded chunk generation, noise, POI seeding, chunk cache by coord.
  - `Ship` — physics (heading, velocity, drift, wind/wave forces), state (HP, cargo).
  - `Wind` / `Weather` / `Time` — global sim, drifting over time.
  - `Camera` — follow + worldspace↔screenspace transforms.
  - `Render` — water layers, terrain, ship, particles, UI/HUD, minimap.
  - `Systems` (later) — Fishing, Combat, Trading, Ports.
  - `Save` — `localStorage` (seed + position + ship + discovered map + inventory).
- **Determinism:** world is a pure function of `(seed, chunkX, chunkY)` so it's
  infinite and reproducible without storing it all.
- **Performance:** only generate/draw visible chunks + a margin; object pools for
  particles; cache static island renders to offscreen canvases.
- **Debug hooks:** expose `window.__HS` (game state, teleport, set weather/wind,
  spawn) so the world can be tested while rAF is paused in a hidden preview tab.

---

## 8. Roadmap

### V1 — "Sailing & Feel" (the slice we build now)
- [ ] Infinite procedural ocean: chunked gen, depth, islands you can see & approach.
- [ ] Ship that feels good: heading + momentum + drift, A/D/W/S, wake.
- [ ] Wind system + compass indicator; sailing with/against wind matters.
- [ ] Animated layered water + waves rocking the boat.
- [ ] Camera follow with lookahead and gentle sway.
- [ ] Day/night + basic weather (clear→cloudy→rain) with color grading.
- [ ] HUD: wind compass, speed, clock; minimap with discovered land.
- [ ] Save/load (seed + position + discovered map) via localStorage.
- [ ] Debug hooks (`window.__HS`).
- **Done = sailing the infinite sea is genuinely pleasant.**

### V2 — Activities ✅ SHIPPED
- [x] Cargo hold (cap) + coin currency.
- [x] Fishing/trawling: cast net (Space), fishing grounds telegraphed by circling
      birds + ripples, biome/time-weighted catch table, net drag, hold limit.
- [x] Ports on ~50% of islands: pier + flag landmark, docking prompt + action.
- [x] Trading: market UI (mouse), per-port produce/demand profiles, prices that
      drift over time → buy-low-sell-high arbitrage. Fish always sellable.
- [x] Port markers on chart; save/load extended (coin + cargo).

### V3 — Danger ✅ SHIPPED
- [x] Storms: full storm weather (big seas, gusts, heavy slanted rain, low vis,
      lightning flash + bolts). Hull stress while in a storm — **reef the sail** to
      ride it out (full sail ≈ 3× the damage of a reefed sail).
- [x] Cannons: mouse-aim, left-click to fire, reload cooldown, arcing cannonballs.
- [x] Sea monsters (serpent): hunt → telegraph (rise, red eyes) → lunge strike +
      knockback; segmented body. Loot: coin + Sea Oil / Scales.
- [x] Pirate ships: approach + strafe + broadside cannon fire. Loot: coin + goods.
- [x] Hull damage, screen shake, damage flash, low-hull warning; sinking sequence
      → respawn at last port (−20% coin, −40% cargo). Docking repairs the hull.
- [x] Enemy health bars, combat-aware prompts, debug spawn/hurt/storm hooks.

### V4 — Depth ✅ SHIPPED
- [x] Ship upgrades (Shipwright tab): hull (HP+cargo), rigging (speed+storm),
      cannons (dmg+reload), nets (catch rate+rarity), spyglass (chart range).
- [x] Contracts/bounties (Harbour tab): delivery to a named port (chart marker +
      auto-complete on dock) and hunt bounties (progress on kills). HUD tracker.
- [x] Ocean regions: Calm Shallows / Open Sea / Coral Reaches / Cursed Deep —
      distinct water tint, catch bias, roughness, and danger/monster rate.
- [x] Procedural audio (Web Audio): ambient sea + wind driven by roughness/wind,
      slow lofi chord pad, combat tension drone, SFX (cannon, hit, catch, coin,
      thunder, upgrade). Mute with **P**.
- [x] Logbook tab: fish caught, pirates sunk, leviathans slain, contracts,
      ports visited, coin earned, ship upgrade summary.
- [x] Day/night cycle slowed to a full 10-minute loop.

### V5 — Legend ✅ SHIPPED
- [x] Crew (Crew tab): 5 roles (Navigator/Gunner/Fisherwife/Cook/Bosun) with
      passive bonuses; berths = 2 + hull level; hire at ports.
- [x] Boss leviathan: 600 HP, 3 phases (tentacle slams → + projectile spew →
      frenzy), big health bar, appears in the Cursed Deep. Great spoils on kill.
- [x] Quest chain "The Deep Song" (Quests tab): dock → land a rare catch → reach
      the Cursed Deep → slay the Leviathan → return its Heart. Reward: coin +
      the Leviathan crest.
- [x] Cosmetics (Shipyard tab): recolour hull / sail / trim and pick a sail crest
      (unlock with coin; Leviathan crest earned via quest).
- [x] Controller support (gamepad: sticks steer/aim, buttons fire/act/map/zoom)
      and touch (virtual joystick + on-screen fire/action/map buttons, tap-to-fire).

### V6 — Wider Seas ✅ SHIPPED
- [x] Weather forecasting: the next weather + time-to-change shown in the HUD;
      Spyglass/Navigator reveal one further ahead; storm-warning banner.
- [x] More enemy variety: Raider (fast rammer), Warship (heavy, triple volley),
      Ghost ship (spectral, night/cursed), Deepmaw (ranged monster). Region-weighted.
- [x] Deeper lore: region descriptions + per-port rumours; drifting **flotsam**
      that yields coin + a lore fragment; 16-entry Sea Lore collection in the Logbook.
- [x] Ship classes: Sloop / Cutter / Merchantman / Frigate — distinct hull, hold,
      speed, turn, reload, size, and cannon count. Pick at the intro; buy/switch
      at the Shipyard. Frigate fires twin cannons.

### V7 — A Living Ocean ✅ SHIPPED
- [x] NPC ships on routines: merchants running port-to-port, fishing boats working
      the grounds, navy patrols, wandering ghost ships — a trafficked sea.
- [x] Piracy: attack any neutral ship to earn **notoriety + a bounty**; navy
      patrols hunt the notorious. Pay off your bounty at a harbour. WANTED HUD.
- [x] NPC captains: named vessels you can **hail** (Space) for rumours/coin;
      captains give the journey quest-lines.
- [x] Wildlife: whales (surfacing + spouts), sharks (dorsal fins), dolphin pods
      (leaping), turtles, gull flocks, and glowing sea-sprites — region-weighted.
- [x] Weather-driven trade events: shortages/gluts (spiked by storms) that move
      regional prices; shown as Market News in the harbour.
- [x] Deeper quest lines (Journeys): multi-step captain quests (Ghost Fleet,
      Naturalist, Merchant Prince) tracked in the Quests tab.
- [x] Achievements: 8 unlockables tracked in the Logbook.
- [x] Controller remapping: a Controls screen (O) to rebind keyboard & gamepad,
      saved to localStorage.

### V9–V14 — The Second Era ▶ see `EXPANSION_PLAN.md`
- The full expansion roadmap: V9 Living Sea (organic islands, reefs/shoals,
  physical roaming storms, icebergs/riptides/whirlpools) → V10 Gunnery & Spoils
  (ammo types, utility shots, loot drops, inventory, boat naming) → V11 Trade
  Winds (region goods, fish depletion, price ledger) → V12 Banners of the Sea
  (factions, territory, NPC-vs-NPC war, forts, cults) → V13 Many Flags (nine
  culture kits) → V14 Your Flag (fleet, stronghold, player faction, diplomacy).

### Parking lot (future)
- Persistent world simulation, leaderboards, more journey chains, seasonal events.

---

## 9. Open Questions / Parking Lot
- ¾ tilt camera vs pure top-down? (Start top-down; revisit.)
- Permadeath vs respawn-with-cost? (Start: respawn at last port, lose some cargo.)
- How "fantasy"? (Start subtle; leviathans + cursed waters, no spellcasting.)
- Crew members as a system? (Parking lot for V4.)
