# Helmsong — Handoff / Continuation Doc

> A lofi, isometric-pixel, fixed-world sailing game. Wind Waker's sailing joy
> crossed with a fishing/trading/combat sim in an 1800s fantasy world. Everything
> is procedural and code-drawn. This doc is a standalone brief so a fresh session
> can continue with no prior context.

## ⚓ STATE OF THE GAME (2026-07-02 — read this first)

**V1–V14 ARE ALL SHIPPED. The V9→V14 expansion plan is COMPLETE** (see
`EXPANSION_PLAN.md` — every phase marked ✅). The game has since been hardened by
**four full QA passes, each with a same-session fix batch (≈37 defects found &
fixed, all re-verified, console clean throughout):**
- `PLAYTEST_V14.md` — systems playtest (7 fixes: ground-drain balance, escort
  respawn, harbour layout, chain-slow visual, fort first-shot grace…)
- `PLAYTEST_NEWPLAYER.md` — fresh-eyes first session (8 fixes: burn-blame
  attribution, Open Sea spawn cadence, contract abandon, quest tracker…)
- `QA_SENIOR.md` — edge/regression battery (4 fixes: **quest-Leviathan spawn
  off-by-one**, wrap-aware stronghold, mid-sink saves, short-window UI scale;
  plus resize self-heal)
- `UX_REVIEW.md` — interface audit (9 fixes: **clickable action bar**, crisp
  720p, chart legend, verb-first buttons, terminology…)

**Most recent work (user-directed):** the ship-name-collision fix (`pickShipName`,
'Wreck of the Amberline') and the **menu rework** — tabbed **captain's screens** on
`I` (Hold · Ledger · Factions · Log · Lore, available anywhere incl. docked), port
slimmed to 6 tabs, and the **always-on-top medallion-pill status widget** (coin ·
hold · ship name). Details in §6's dated entries. **A fifth QA pass (play-feel run
of the reworked menus + new-player flow, 2026-07-02) followed — 5 findings, all
fixed & verified same-session** (see the PLAY-FEEL PASS entry in §6).

**LAND SPRITES RETIRED → CODE-ART VARIETY PASS SHIPPED (2026-07-03,
v0.17.4–v0.18.2, 4 verified batches, console clean).** The FLORA sprite batch
(v0.17.0–0.17.3) was wired, verified — and then **rejected by the user in
play** ("way too detailed, look just wrong, the keying messed stuff up — go
back to just the simple code assets and refine those"). v0.17.4 empties
`LAND_SPRITES` (the `drawSpriteBuf`+`sprTint` buffer pipeline remains,
dormant) and removes `assets/sprites/`. Then the code art itself was upgraded:
**flora v2** (hash-picked silhouettes per biome: oak/cloud/poplar, curved
frond palms, gnarled dead forks, stacked snow pines, saguaros w/ arms+blooms),
**port hamlets** (1–2 shore huts per port + dockside crates/barrels + building
polish per culture), **sea stacks v2** (crooked faceted stone spires). Details
in the dated §6 entry. **RULE: Helmsong's art stays code-drawn** — generated
sprites have now been retired twice (ghost ship, land assets). The game is
**v0.18.2**, a git repo (log = changelog), with 4 save slots.

**What's next (pick with the user):**
1. ~~**Play-feel pass**~~ — DONE 2026-07-02 (menus/new-player flow; 5 fixes, §6).
2. **Post-plan backlog** (biggest first): automated fleet trade routes (passive
   income, raidable); sieges AGAINST your stronghold / faction fort captures;
   remaining V13 culture kits (Mayan/African/Greek/Egyptian/Roman/Japanese — each
   is CNAMES + drawPortHouse branch + VESSELS hull on the existing scaffold);
   NPC hazard parity (npcs ignore shoals/slicks/flows); relations drift;
   per-culture cult monsters; music flavour per culture.
3. **Portfolio pass** — trailer + case study like SWARM's (the reusable
   canvas-capture rig lives in the ACES kit; run it from /tmp).

---

## 1. Where things are / how to run it

- **Game:** one self-contained file — `~/Desktop/Helmsong/index.html`
  (HTML5 Canvas 2D, vanilla JS, no build step, ~5,400 lines).
- **Docs (all in `~/Desktop/Helmsong/docs/`):** `GAME_DESIGN.md` (original pillars,
  V1–V7 roadmap) · `EXPANSION_PLAN.md` (V9–V14, all ✅) · the four QA reports listed
  above · this file.

**Run / preview it** (Desktop paths 404 in the preview sandbox, so serve a copy from `/tmp`):
1. `cp -f ~/Desktop/Helmsong/index.html /tmp/helmsong-preview/index.html`
2. Preview server config already exists in `~/.claude/launch.json` as **`helmsong`** on **port 8801** (`python3 -m http.server 8801 --directory /tmp/helmsong-preview`).
3. Start it with the `preview_start` tool (name `helmsong`), click **Set Sail**.

**Standard dev loop used throughout this project:** edit `index.html` → copy to `/tmp/helmsong-preview/` → reload preview → drive the game via `window.__HS` debug hooks + screenshots → check `preview_console_logs` for errors → `window.__HS.reset()` to clear the save and start clean. There has been **zero tolerance for console errors** — every change was verified clean.

---

## 2. Architecture (all in one file, in-order sections)

- **Pixel buffer render:** world is drawn to a low-res offscreen canvas `buf` (BW×BH ≈ window/PIXEL) then integer-upscaled nearest-neighbor to screen for the chunky pixel look. HUD is drawn crisp at full res on top.
- **Projection:** 3/4 isometric — `proj(wx,wy,wz)` squashes Y by `ISO_Y` (0.58) and lifts height by `ISO_Z`. Screen↔world is linear per axis (no rotation).
- **Zoom vs chunkiness are separate:** `viewSpan` (world units visible) is the zoom (mouse wheel); `PIXEL` is screen-px per buffer-px (adapts to zoom so far-out isn't mush).
- **World (V8 — finite, fixed, wrapping):** a bounded, always-identical world of `WORLD_W`×`WORLD_H` = 80000×64000. It **wraps east–west** (content is periodic: `genChunk` DECIDES content by `wrapCol(cx)` but POSITIONS at continuous `cx`, so sailing `WORLD_W` east re-enters the identical map with **zero seam and no wrapped-distance math** — coordinates stay continuous). It's **bounded north–south** by impassable polar ice (`clampPole(ship)` keeps `y∈[0,WORLD_H]`). `genChunk(cx,cy)` (CHUNK=1000) still makes islands / grounds / flotsam, cached in `chunkCache`. New games start at `START` (north-temperate) via `findOpenWater()`.
- **Continents:** 3 fixed large landmasses (`CONTINENT_DEFS`, `buildContinent`, `contCache`) at mid-latitudes, **injected into every chunk they overlap** (`continentsInChunk` inside `genChunk`) so land collision / ports / docking / rendering pick them up for free; render de-dupes them by object identity. They repeat every `WORLD_W` like everything else.
- **Regions (latitude climate):** `regionAt(x,y)` is now **temperature-by-latitude** — hot at the equator (`y=EQUATOR`, middle), cold at the poles (top/bottom) — plus a PERIODIC longitude field (sampled on a circle so it repeats every `WORLD_W`) that waves the band edges and places the Cursed Sea. Bands: ice(poles) → calm/temperate → tropic → desert(equator). Big, coherent, no more snow-in-desert. `REGIONS{}` holds tint/danger/biome/fish/lore/gate per region.
- **Entities:** `ship` (player), `enemies[]` (hostile), `npcs[]` (neutral traffic), `wildlife[]`, `shots[]`, `fx[]`. All collide with land via `keepOffLand()`.
- **Loop:** fixed-timestep `update(STEP)` inside `frame()`; `render()` once per frame. `window.__HS.tick(dt,n)` steps+renders while rAF is paused (for headless testing).
- **Save:** `localStorage` key `helmsong_v1` (autosave every 4s). Keybinds saved separately as `helmsong_binds`.

---

## 3. What's built & working (V1 → V7, all verified, no console errors)

- **V1 Sailing:** wind + waves + weather + day/night + camera + HUD + minimap.
- **V2 Activities:** fishing/trawling (grounds telegraphed by birds/ripples), ports with markets, trading with per-port produce/demand price sim, cargo + coin.
- **V3 Danger:** storms (lightning, hull stress — reef sail to survive), cannons (mouse-aim, click), sea serpents, pirate ships, hull damage → sink → respawn.
- **V4 Depth:** ship upgrades (Shipyard), contracts/bounties (Harbour), ocean regions, procedural audio (Web Audio; **P** mutes), Logbook stats.
- **V5 Legend:** crew (5 roles, passive bonuses), boss **Leviathan** (3-phase, in the Cursed Deep), quest chain "The Deep Song", ship cosmetics, controller + touch input.
- **V6 Wider Seas:** weather forecasting, more enemy variety (raider/warship/ghost/deepmaw), lore fragments via flotsam, 4 ship classes (Sloop/Cutter/Merchantman/Frigate).
- **V7 Living Ocean:** NPC ships on routines (merchant/fisher/patrol/ghost), piracy → notoriety + bounty → navy hunts you (pay it off at a harbour), named captains you can hail, wildlife (whale/shark/dolphin/turtle/gull flock/sea-sprite), weather-driven trade events, captain "journeys" (deeper quest lines), 8 achievements, controller remapping.

### Post-V7 polish & tuning (most recent work)
- **Lantern** (key **L**): warm light pool at night; +30% night fishing but attracts more danger; upgradeable; `waterGate`-aware. Night was lightened (brighter floor).
- **Playtest fixes:** Cursed Deep made common + findable; **region-tinted chart with port-name labels + legend**; **on-screen guide arrows** (⚓ port, contract targets, Cursed Deep); easier docking (bigger radius, no speed gate); calmer seas + safe havens + grace period; softer respawn penalty. (Also caught+fixed a bug where a too-large "safe zone near ports" silently blocked ALL enemy spawns.)
- **Backlog batch:** wind now base-propelled (never becalmed) + top-speed cap; **land collision for all entities**; fewer enemies / more neutral traffic; **economy rebalanced** (quadratic upgrade costs, narrower trade margins) so you can't buy everything in 10 min; **2 new regions Ice (Frostreach) + Desert (Sunspine Flats)**; **soft water-gates** (ice needs Icebreaker Hull, cursed needs Warded Keel — enter anyway but −5hp/s & half speed until you buy the component); **biome islands** (green/tropic/dead/ice-snow/desert-cactus) with varied sizes; **map port names**.
- **Day/night fix:** cycle 10→14 min AND day-biased (`dayFactor = pow(raw, 0.5)`) so the dim portion dropped from ~50% to ~33% of the cycle (fixes "night all the time").
- **Spatial ocean colour (OSRS-style):** water colour is no longer a full-screen filter. `drawWater()` samples `regionAt` over a 64×38 grid mapped to the visible world, **box-blurs the low-res colour field** (`blurField`, r=4), then bilinear-upscales it → smooth, world-anchored ocean gradients; you SEE the ice/cursed/coral zones as coloured patches that blend at their edges. Gated seas are strongly coloured (Frostreach pale-blue, Cursed Deep violet) so hazard water is obvious from afar.
- **Visual-variety pass:** every ship type/class now has its own silhouette + rig (data-driven `VESSELS{}` + `drawVesselBody`/`drawVesselRig`/`sqSail`), a new **kraken** grappler sea monster (`drawKraken` + `lurk/reach/grab` AI in `stepEnemies`), and polished wildlife sprites (`drawWild`). See §6 for detail.
- **UX/QoL fixes (most recent):**
  - **HUD overlap** — the contract tracker anchored to a fixed y and collided with the taller weather panel (2 forecast lines w/ spyglass); now anchors at `16 + panelH + 8`. Added `hudTopAvoid()` so world guide-arrows/labels stay out of the top-left/right HUD columns.
  - **Stuck-key fix** — reefing sail to 0 then opening the map could leave a movement key "stuck" (missed keyup), pinning the sail (W & S cancel) so the boat couldn't move. Added `clearKeys()` wired to window **blur** and every map/port/settings/pause transition; genuinely-held keys re-assert via OS key-repeat.
  - **Bounty payoff** — `payBounty()` now takes **partial payments** (pay what you can, chip it down); the option is a prominent banner at the **top** of the Harbour tab (was buried at the bottom, full-payment-only). New coinless escape loop: **sinking pirate ships eases your bounty by 15** (clears notoriety + calms the navy at 0). First-piracy toast points to the Harbour.
  - **Sell all** — market rows now show a **Sell all** button (only when you own >1) next to Buy/Sell; `doSellAll()` dumps the whole stack in one click.
- **Two-mode map (scroll to zoom):** the open chart (M) now has a ship-centered **LOCAL** mode and a whole-world **WORLD** mode; **scroll** zooms between them (out past `MAP_LOCAL_MAX` flips to world; scroll in returns). `drawMinimap` dispatches to `drawLocalChart`/`drawWorldChart`; the world chart is rendered once to a cached offscreen canvas (`worldChart()` — climate bands + continents + ports over one full period) and blitted, with the player marker/contract pins drawn live on top. `toggleMap()` always opens in local mode. Wheel/gamepad-zoom route to the map when it's open. State: `mapRange`, `mapWorldView`.
  - **World-chart extras:** **continent name labels** (Norhaven/Sundara/Sudreach — `CONTINENT_DEFS[].name`), a **"you are here" readout** (region + pseudo-latitude via `latLabel(y)`: 90°N pole → 0° equator → 90°S), and **click-to-set waypoints**. `mapClick()` inverts the click via the stored `mapGeom` (world or local) and drops a `waypoint {x,y}` (world-mode clicks snap to the ship's nearest period so you steer the short way). The waypoint shows a pulsing crosshair on both charts + minimap (`drawWpMark`), a `Waypoint` guide arrow in-world (`drawGuides`), auto-clears within 200u (`stepDanger`), and clears by clicking its marker or right-click. Debug: `__HS.waypoint` / `__HS.setWaypoint(x,y)`.
- **Custom sprites — THREE pipelines, ALL land use retired (2026-07-03):**
  - **Buffer sprite** (`drawSpriteBuf(key,wx,wy,wz,worldH,pivY)`): draws INTO the pixel buffer (`bctx`) at the call site, so props would pixelate with the terrain AND depth-sort against ships for free. Day/night via `sprTint` (cache of pre-darkened/desaturated copies). **DORMANT** — `LAND_SPRITES{}` is empty since v0.17.4 (user retired the generated land sprites after seeing them in play; the wired versions live in git v0.17.0–v0.17.3). A few call sites remain (fort/berg/flotsam/loot) and no-op; the code art is canonical.
  - **Single flat sprite** (`SPRITES`/`SPR`, `drawSprite()` → `spriteQueue` → `flushSprites()` crisp after the buffer blit with day/night `filter`). **Only for NON-rotating iso assets** (islands, ports, props). A flat top-down PNG rotated in 2D looks broken on the 3/4-iso camera — do NOT use it for ships.
  - **Directional sprite** (`SPRITES_DIR`/`SPR_DIR`, `drawDirSprite()`): 8 pre-rendered ISO frames per object (bow facing E,SE,S,SW,W,NW,N,NE in that array order = sectors 0..7); picks the frame nearest `heading` and draws it **un-rotated** → keeps the real 3D-iso look for things that turn. **This is the way to do custom ships/monsters.** Proven with the **green ghost ship** (`ship_ghost_r{r}c{c}.png` × 8, sliced/keyed/recolored from a Nano-Banana 3×3 sheet) — but the user later **retired it** ("bring back the style we had before"): the `sprDir/sprLen/sprPivot` keys were removed from `VESSELS.ghost` and the `SPRITES_DIR.ship_ghost` entry is commented out, so the ghost is **code-drawn (tattered rig) again**. The pipeline itself (`drawDirSprite` + the `drawEnemyShip` branch) is intact and the 8 PNGs remain in `assets/`; re-enable per the comment at `SPRITES_DIR`. Frame index = `((round(heading/(π/4))%8)+8)%8`.
  - Asset processing lives in throwaway Python (PIL): flood-fill/global key the baked checkerboard → alpha, optional hue-shift recolor, slice grid, resize uniform, save to `assets/` (Desktop + `/tmp/helmsong-preview`).
- **More vessel variety (code-drawn, bigger ships):** added 4 new `VESSELS` silhouettes + full spawn/AI/loot wiring — **galleon** (large 3-mast pirate/treasure ship, gunports+sterncastle; enemy via `spawnEnemy`/`pickEnemyType`, big purse + pearls/rum loot), **brig** (fast mid 2-mast pirate; enemy), **trader** (fat high-castled merchant carrack; NPC via `spawnNpc`/`pickNpcType`, `rich` hold → pearls/spice), **manowar** (navy ship-of-the-line, biggest; NPC navy — spawns near ports rarely and as the notoriety-hunter when bounty>75). Loot/toasts in `killEnemy`/`killNpc`, heavy cannon dmg in `enemyFireA`.
- **Ship code art enriched** (for the still-code-drawn vessels): standing **rigging ropes** (masthead→bow/stern/rails) + **deck planking** in `drawVesselBody`/`drawVesselRig`, all types, full 3D-iso + dynamic sails.
- **No vessel pick at start:** the intro's "choose your vessel" cards are gone (removed `#shipPick` + `buildShipPicker`/`pendingClass`); every new game begins in the **Sloop**. The Cutter/Merchantman/Frigate are earned by buying them with coin at the **Shipyard** (`switchShip`), unchanged. (If you want stricter unlock gating — e.g. require hull-upgrade tiers before a class is purchasable — that's a `switchShip`/Shipyard-tab change, not done yet.)
- **Pause menu + New Game:** Esc now opens a real **pause menu** (`drawPause`/`menuBtn`/`pauseButtons`, mouse-driven like the port/settings screens) — Resume · Controls · Quit to Title (saves + reloads) · New Game (→ `ui.pauseConfirm` guard → erase save + reload). Intro screen shows **Continue Voyage + New Game** when a save exists (else just Set Sail); `start(fresh)` skips the load when starting new. Save key unchanged (`helmsong_v1`).
- **V8 — Designed World (biggest rework; see §2 "World"):** replaced the infinite noise world with a **fixed, finite, always-identical** map that **wraps E–W** and is **bounded by ice poles**, sized so a pole-to-pole crossing is ~7 min. Biomes are now **latitude-based climate bands** (cold poles → temperate → tropic → hot desert equator) — big and coherent instead of tiny/incoherent. Added **3 large fixed continents** at varied latitudes (2 temperate-green, 1 lush tropic). Verified: seamless circumnavigation (sailing +WORLD_W returns the identical island), impassable poles, continent collision + dockable ports, all 6 regions present incl. a findable Cursed Deep, no console errors. **NOTE:** world coordinates changed — old saves must be reset (the save was reset).
- **Content batch — 2 new ships, 2 hazard regions, 2 sea monsters (all verified, console clean):**
  - **New vessels (code-drawn):** **corsair** (`VESSELS.corsair`) — sleek low xebec, 3 raked masts, ram prow, long bowsprit, crimson sails; fast enemy pirate (hp72, speed134, quick light volley). **dread** (`VESSELS.dread`) — multi-deck pirate **flagship**, 4 masts + gunports + sterncastle, biggest silhouette (`big:1.5`); rare heavy enemy (hp320, 4-gun volley @15dmg, purse 210 + pearls/rum/scales). Both fully wired: `spawnEnemy`, `pickEnemyType`, `killEnemy` loot/toasts, `enemyFireA` heavy-dmg list. **Fixed a latent bug:** dread/galleon used `region.danger>1.3`/`>1.0` which **no region satisfied**, so they never spawned — retuned to `>=1.0` (dread rare ~3.5%, galleon ~7.6% of Open-Sea/Verdigris spawns).
  - **New hazard regions (use the existing `waterGate` scaffold):** **`sargasso`** — *Sargasso Tangle*, murky-green warm water with drifting **kelp fronds** (`drawKelp()`, only drawn when `curRegion==='sargasso'`); gate `weedcutter` **Weedcutter Ram** (560c). **`toxic`** — *The Verdigris*, fetid yellow-green murk; gate `sealed` **Sealed Hull** (720c). Both added to `REGIONS`, `REGION_COL` (chart), and placed in `regionAt()` via **one extra periodic `haz` fbm field** (sampled only in mid-temp bands, so ~+1 fbm/cell). Gate = −5hp/s (net ~−3/s vs repair) + half-speed until you fit the component; components auto-list in the Shipyard (it iterates `UPGRADES`) — **had to shrink the upgrade row height 38→31** so 10 rows + cosmetics still fit the port panel.
  - **New sea monsters:** **`angler`** — lantern-lure ambush predator: `lure`→`chomp`→`dive` AI, drifts submerged behind a glowing bait-orb (drawn toward the ship), surges for an 18-dmg bite; red eyes + toothy maw open in `chomp`. Attracted to lantern-light; spawns in monster/deep water (`drawAngler`). **`jelly`** — *Jelly Bloom*: slow-drifting cluster of 6 glowing translucent bells + violet tentacles that **shock-pulses (~6dmg/1.3s)** ships lingering inside it; a hazard you sail around, spawns in warm `biome:'tropic'` water incl. the Sargasso (`drawJelly`). Both wired into `drawEnemy`, `stepEnemies`, `spawnEnemy`, `pickEnemyType`, `killEnemy`. Debug: `__HS.spawnType('corsair'|'dread'|'angler'|'jelly')`; find hazard water by scanning `regionAt` (sargasso ~1.5% / toxic ~0.7% of the sea).
- **Ornate HUD/menu re-skin — now uses the real PixelHUDUI asset pack (full-ornate, sprite-based):** user reference was a Graveyard-Keeper-style ornate fantasy HUD. **Two iterations:** (1) first pass recreated the style *code-drawn* (restrained brass/teak); (2) user then asked for **full ornate "just use the asset pack"** + a **circular minimap**, so the chrome helpers were **rewired to draw the actual pack PNGs** (code-draw kept only as a fallback until images load).
  - **Assets:** copied the pack's 8 sprites into `assets/hud_*.png` (also `/tmp/helmsong-preview/assets/`), loaded via `HUDART`→`HIMG` (mirrors the `SPRITES`→`SPR` loader). Source: `~/Desktop/PixelHUDUI_v1.0_FreeDemo/` (indigolay.itch.io) — **ReadMe says "Free for personal & commercial use"**; credit them if shipping publicly. Full kit (82 sprites, $) has ornate banners/portraits/more bars if wanted later.
  - **How it's wired (all in the chrome helpers near `bar()`/`roundRect()`):** `img9(img,x,y,w,h,b)` = 9-slice (corners fixed, edges/centre stretched, nearest-neighbour); `imgHBar(img,x,y,w,h,cap)` = horizontal-slice for thin bars (crisp end-caps + stretched middle, squashed to height). **`chromePanel`** now 9-slices `HIMG.slot` (UI_Slot_Selected, border 12; thin strips 9) → every panel/strip is the ornate gold frame; `tint` option overlays a colour (WANTED red / heart magenta). **`chromeBar`** draws `HIMG.barBg` (UI_Progress_Style2) via `imgHBar` + a coloured glossy fill inside (frame drawn FIRST since its centre is opaque). **`brassButton`** 9-slices the slot + a gold/danger gloss overlay on hover/active → all `drawBtn`/`drawTabBtn`/`menuBtn`/settings-rows inherit it. `HIMG.heart` (UI_Indicator_Heart) drawn on the HULL bar. `UICOL` retuned to the pack's exact golds (`#f1db97`/`#ceb272`/`#704f29`, wood `#4c2327`) so code-drawn bits (compass ring, circular minimap ring) match the sprites.
  - **Circular minimap:** the corner minimap is now a **circle** — `drawMinimap()` branches: closed = clip `drawLocalChart` to a circle + `goldRing(cx,cy,R)` (gold gradient annulus + dark keylines + 4 cardinal rivet-studs, matching the pack gold); open (M) = still a rectangular `chromePanel`-framed full chart. Bottom-left **compass** is also a gold ring (`drawCompass`), so the two corners mirror.
  - **Layout tweaks for the chunkier frames:** status panel grew to 196×104, bars are now 16px tall (`chromeBar` needs height for the sprite frame), and the left-column strips (coin/region/WANTED/heart) shifted down + widened to 196 + taller (26–28px) so the ornate frame leaves a dark inner. Verified error-free: gameplay HUD, port (panel/tabs/Buy-Sell), pause, controls, circular minimap, open chart, fresh game.
  - **To retune:** everything funnels through `chromePanel`/`chromeBar`/`brassButton` + `UICOL` + the 9-slice borders. `chromeBanner`/`ribbonPath` (notched ribbon) still built-but-unused if a location banner is wanted.
- **UI polish pass — font, leather panels, button padding (all verified, console clean):**
  - **Font → VT323** (user picked from an in-game 3-way demo of Pixelify Sans / IM Fell English / VT323; shown via a `show_widget` comparison since preview screenshots aren't user-visible). Loaded via a Google-Fonts `<link>` in `<head>` (all 3 families still linked). **Every canvas font string is `…px VT323,ui-monospace,monospace`** — swapping the whole game's font is a single find-replace of `VT323` (the family is unquoted-multiword, valid in canvas; `ui-monospace,monospace` is the offline fallback). Bumped the smallest sizes +1px (8/9/10→9/10/11) for legibility. **Known quirk:** VT323's capital **M** reads a little like **N** at ≤12px ("Map"→"Nap", "Market"→"Narket") — inherent to the font; larger sizes or a different family fixes it if it bothers.
  - **Leather menu backgrounds (replaced the stretched pixel-texture):** `chromePanel`/`brassButton` no longer draw the slot sprite's stretched centre. New `leatherFill(x,y,w,h,r)` = vertical brown gradient + a generated grain tile (`leatherTile()`, transparent specks, made once) + a soft radial vignette + top sheen; then `img9(HIMG.slot,…, true)` draws **frame only** (new `noCentre` arg on `img9`). So panels/buttons are now solid dark leather + crisp gold frame. `tint` still overlays for WANTED/heart. To lighten: edit the `leatherFill` gradient stops.
  - **Button padding + auto-sizing:** thinned the button frame (9-slice border 11→9 = more inner room). New `btnW(label,fontPx,padx=10)` = `textWidth + 2*(padx + BTN_FRAME(8))`. Applied to the tight buttons so text has ~10px horizontal breathing room: port **tabs** (auto-width, flow left with 7px gaps, h30), **Buy/Sell/Sell-all** (market columns shifted left to free room, right-aligned auto-width, row pitch 22→24), shipyard **Buy**, **Hire**/**Take** (crew/quest/contract), **Set sail**. Dense market rows cap button height at 20 (can't fit full 6px vertical padding + frame in a 24px row); roomy buttons (pause 240×40, settings key-rows) were already spacious and left as-is.
- **Bottom action bar (Graveyard-Keeper-style hotbar — matches a user reference image):** new `drawActionBar()` draws a centred leather panel at screen-bottom containing: **SAIL gauge** (orange, left) · **compass** (raised, straddling the top edge — `drawCompass` was refactored to `drawCompass(cx,cy,R,showLabel)`) · **clock plaque** (HH:MM + day/night) · **SPEED gauge** (blue, right) across the top; a **6-slot hotbar** below (net/cannon/lantern/anchor/chart/mute — `ACTION_SLOTS`, each = full slot-sprite + code-drawn `actionIcon()` + a hotkey chip from `keyCap()` reading live `bindK`; active toggles glow gold via `sl.act()` — lantern/anchor/map/mute, plus a red slash on the mute icon when muted); and a long **HULL/health bar** (red, with the heart) at the bottom. Sized `barW=clamp(W*0.52,500,700)` so it never collides with the bottom-right circular minimap. Hidden when `ui.mapOpen || ship.sinking || ui.port`.
  - **HUD restructure this caused:** the **old top-left SAIL/SPEED/HULL panel was removed** (those gauges now live in the action bar) and the **bottom-left compass moved** into the bar; the coin/region/WANTED/heart strips shifted **up to y=16+** to fill the freed top-left; the context prompt + catch-toast moved **above** the bar (`H-122-12-52` / `-78`). Helmsong has **no item/inventory system**, so the "hotbar" is an **action/hotkey bar**, not collectible items — slots are display-only (not clickable, to avoid stealing the click-to-fire-cannon input). NOTE for touch: `drawTouchUI()` (virtual joystick + buttons) may now overlap the action bar on touch devices — untested, revisit if targeting mobile.
- **UI text scale (v2 — replaced the whole-HUD scale after user feedback "elements way too big, I just wanted the text bigger"):**
  - **First attempt (kept as infrastructure, OFF by default):** a resolution-based whole-HUD scale — `render()` draws HUD+menus inside `ctx.scale(US,US)` with a temporary `W,H`→`UW,UH` swap; `hit()` and `mapClick()` divide the mouse by `US`; `drawGuides()`/`drawTouchUI()` draw unscaled outside the block. Now `computeUIScale()` just sets `US = clamp(uiScaleMul,0.5,4)` (**=1 default, so chrome is native-size**); `__HS.uiScale(m)` still scales everything if ever wanted.
  - **Current approach — FONT scale:** `FONT_SCALE = 1.8` + `fnt(n)` → `round(n*1.8)+'px VT323,…'`. **Every canvas font call goes through `fnt()`** (all ~95 former literal strings were replaced — grep `px VT323` should only hit the `fnt` definition). `btnW()` measures with `fnt()` so **auto-sized buttons grow with the text**. Tune live with `__HS.fontScale(m)` (default 1.8) — but layout is hand-fit to ~1.8, so big deviations need layout retouches.
  - **Two-tier typography (user request — "small description text should be a sans like Geist"):** added **`fntS(n)`** = `'500 '+round(n*1.3)+'px Geist,-apple-system,"Segoe UI",sans-serif'` (Geist added to the Google-Fonts `<link>`; weight 500 for legibility on dark leather; the ×1.3 sizing makes `fntS(n)` read visually like `fnt(n)`). **Rule: VT323 (`fnt`) for titles/names/numbers/buttons/labels; Geist (`fntS`) for small descriptions & dim secondary text.** Converted: port header region-lore + rumor lines; shipyard "HULL — …" line, upgrade descs, OUTFITTING header; market column headers; weather forecast rows; contract tracker (top-right); crew headers + bonus descriptions; quest reward line + JOURNEYS header; harbour section headers + WANTED banner header; logbook SHIP/ACHIEVEMENTS/SEA-LORE headers + achievement/lore rows; settings subtitle + KEYBOARD/GAMEPAD headers; pause "Esc to resume" + confirm subtitle; open-chart header + legend + "you are here"; clock-plaque day/night word.
  - **Button label polish (user request — "uneven padding, hard to read"):** `brassButton`'s sprite branch now (1) uses a **height-relative frame border** `bb = clamp(round(h*0.27), 5, 9)` so small buttons (Buy/Sell h26 → bb 7) keep a full leather face instead of the label overlapping the gold bevel; glow inset follows `bb`; (2) **optically centres the label** — VT323 renders low on a middle baseline, so the label draws at `y+h/2 − max(1, round(fontPx*0.09))` (font px parsed from the font string); (3) adds a **1px dark drop-shadow** under the label (skipped on active/gold state) + slightly brighter cream `#f4ead0` → labels pop off the frame. Applies everywhere brassButton is used (market/tabs/settings/pause).
  - **Menu polish round 2 (user requests):** (1) **leather seam fixed** — `leatherFill`'s "top sheen" was a hard-edged translucent rect (`fillRect(x,y,w,h*0.38)`) that read as a gradient cut across tall panels; now a linear gradient fading to transparent over the top 55%. (2) **Edge padding — final: 64px** (user escalated 20→32→"make it 64"): port panel content margin is **64px** all round (title py+32, sub-lines py+64/84, tabs py+108 h36, content `ay=py+158`, `ax=px+64`, `aw=pw-128`, right-aligned at `px+pw-64`, Set sail at `py+ph-60`); panel widened 840→**900** so content width stayed ≈constant. Settings got **48px** sides (proportionate for the smaller dialog), pw 620→660, rows from py+88 pitch 29, footer at ph-58. Vertical re-fits to absorb the lower content start: market pitch 28, shipyard cy0=ay+20 / ry=cy0+38 / upgrade pitch 30 / cosmetics pitch 28. (3) **Logbook text bump** — stats `fnt(13)`/pitch 32, SHIP line `fnt(13)`, section headers `fntS(11)`, achievement/lore rows `fntS(12)`/pitch 25; plus the **global sans multiplier raised 1.3→1.4** (fntS(10)=14px) so all description text is a touch bigger. NOTE: port tabs are budgeted to the pixel at ph=700 — any future row additions to market (16 rows) or shipyard (10 upgrades + 4 cosmetic rows) need a pitch trim or taller panel.
  - **Compass moved (user request):** removed from the action bar's centre. Compass now drawn **bottom-left at chart size** — `drawCompass(91, H-91, 71)` mirrors the bottom-right circular minimap (size 150, R 71). `drawCompass` needle/arrow/centre-dot now **scale with R** (were fixed-px for the old R=44). Debug coords text moved 16→x178 to clear it. NOTE: the touch-UI virtual joystick idles at (94,H-94) — visually overlaps the new compass on touch devices (untested, as before).
  - **Action bar restructure (user request — "health bar to the top, more prominent; SAIL/SPD/HULL labels bigger"):** `barH` 122→**142**. Layout now: **TOP = full-width HULL bar** (h18, brighter red `#d64a30`, `HULL` label in `fnt(12)` `#e8765a` + 18px heart icon left of the bar) → hotbar slots (by+42) → **BOTTOM row = SAIL gauge | clock plaque (124×32 inline, no longer straddling) | SPD gauge** (barY by+107, labels `fnt(12)`, measured for placement). Context prompt/toast offsets updated to `H-142-12-44`/`-76`.
  - **Port header text scale-up (user request):** title `fnt(18)` py+26, region-lore + rumor `fntS(12)` py+64/88, coin `fnt(15)`, hold `fnt(11)`; tabs → py+112, content `ay=py+160`. **Market row pitch is now ADAPTIVE** — computed from the real panel height (`clamp(floor((footY-(ay+24)-26)/15), 24, 28)`) so the 16-row list never collides with the Set sail footer on short windows (panel caps at `H-24` below 724px-tall windows; the fixed pitch overflowed at H=700).
  - **Layout retouches for the bigger text** (chrome grew only where text needed room): top-left strips 196→240 wide / taller+restacked (coin y16 h32, region y54 h30, WANTED y90 h30, heart y90/126); weather panel 166→230 wide, `panelH=82+fc*22`, rows every 22; contract tracker pitch 22; `hudTopAvoid` updated to match; action bar — SAIL/SPD labels place bars via `measureText`, clock plaque 124×32, hotkey chips/HULL label use small `fnt(8)`; context prompt h34 (sits at `H-122-12-64`), toast at `-96`; port panel 724×588→**840×700**, header/title spacing reflowed (title `fnt(15)` at py+12, tabs y py+84 h36, content ay=py+132, Set sail h36); market pitch 30 + buttons h26 + price columns at aw-380/-320/-260; shipyard class buttons via `btnW` h28, upgrade pips moved ax+150→ax+210 (long component names), desc `fnt(9)`, pitch 33, cosmetics swatches at ax+90 pitch 30; crew/quest/harbour rows ~28–32 + Hire/Take/Pay h26–28; logbook pitches 21–28 (achievements/lore `fnt(10)`); settings 620×640, rows every 30, bind buttons 172×26, footer via `btnW` h30, title `fnt(14)`; pause `menuBtn` 300×44 spacing 54, PAUSED `fnt(26)`, confirm-dialog y reflowed. Verified at 1280×720: HUD, market/shipyard/crew tabs, settings, pause all fit; tab click hit-test OK; console clean. (Preview screenshot tool had a stuck capture surface after a 1920 resize — page-side dims were correct; HUD is corner-anchored so resolution only moves anchors, not density.)

---

## 4. Debug API — `window.__HS`

Exposed for testing (call from the browser/preview console or `preview_eval`):
`ship, player, world, wind, weather, timeOfDay, cam, enemies, npcs, wildlife, marketEvents` (live state) ·
`teleport(x,y)`, `setTime(t)`, `setWeather(s)`, `setWind(dir,str)`, `setZoom(span)`, `setPixel(n)` ·
`nearestPort()/nearestIsland()/nearestGround()/nearestFlotsam()`, `region()`, `dockAt(isl)` ·
`give(n)`, `fill()`, `upgrade(k)/maxUpgrades()`, `setShip(cls)`, `cosmetic(k,i)`, `crewAdd(r)` ·
`spawn(type)/spawnType(t)/spawnNpc(t,hostile)/spawnWild(t)/spawnBoss()`, `hurt(n)`, `killAll()`, `fire()`, `storm()`, `makeEvent()` ·
`quest()/questSkip()`, `journey(id)`, `bounty(n)`, `lantern(bool)`, `forecast()`, `mute()`, `openSettings()`, `binds()`, `touchUI(on)` ·
`tick(dt,n)` (step sim+render while rAF paused — key for headless testing), `save()/load()/reset()` ·
data: `GOODS, UPGRADES, SHIPS, REGIONS, CREW, COSMETIC, QUEST, JOURNEYS, ACHIEVEMENTS, LORE`.

**Added V9–V14 (all still live):**
`storms, spawnStorm(x,y), bergs, spawnBerg(x,y)` (V9) ·
`loot, spawnLoot, lootSpill(x,y,coin,{id:qty}), nameBoat(n)` (V10) ·
`AMMO, giveAmmo(id,n), setAmmo(id), openInv(), slicks, chums` (V10) ·
`groundStock, stockHere(), drainHere(amt)` (V11) ·
`FACTIONS, rep(), setRep(f,v), waters(), warNow(), startWar(x,y), endWar(),
fortState, nearestFort(), damageFort(n)` (V12) ·
`CULTURES, culture()` (V13) ·
`escorts, fleet(), giveEscort(cls), hold(), vault()` (V14) ·
`uiScale(m), fontScale(m)` (UI). Top-level (reachable from console, not on __HS):
`shots, grap, war, splashUtility, ui, player, portFaction, portCulture, pickShipName`.

**Testing gotchas (learned the hard way):** the preview tab runs rAF LIVE between
`preview_eval` calls — run multi-step scenarios inside ONE eval; `dockAt()` leaves
the port OPEN (pair with `closePort()` — port pauses env drains & sim); `tick`
updates before render so `curRegion` is stale for one tick after `teleport` (tick
twice); emulated viewport changes may not fire a resize event (the game self-heals
per-frame now); autosave is real-time-throttled — hidden tabs may not autosave, use
`__HS.save()` explicitly in tests.

---

## 5. Key tunables (search these to rebalance)

- Physics: `stepShip()` thrust `92 + 155*windPush*effWind`, speed cap `172*shipC().spd`, drag exp values.
- Combat: `ENEMY_CAP=2`, spawn rate in `stepEnemies` `(11+rand*11)/(danger*lure)` + `region.danger*0.6*lure`, `graceT` (safe window), `nearbyPortWithin(320)` safe radius, enemy `reload/fireRange/volley` in `spawnEnemy`.
- Traffic: `NPC_CAP=11`, `WILD_CAP=18`, `pickNpcType()` (patrols capped at 2).
- Economy: `UPGRADES[].cost` (quadratic), `priceFor()` margins, `killEnemy/killNpc` loot, `switchShip` prices.
- World size/shape: `WORLD_W`/`WORLD_H` (80000×64000 — bump for a bigger/slower world), `START` (spawn latitude), `clampPole` (pole walls). Cruise speed ≈ 172–204 u/s, so ~7 min per 64000.
- Regions/water: latitude bands in `regionAt()` (the `temp<…` thresholds set band widths; `EQUATOR`), the periodic `lon`/`cur` fields (band waviness + Cursed Sea), `REGIONS{}` (tint/danger/biome/gate), `BIOME{}` island palettes, `drawWater()` grid `GW/GH` + `blurField` radius.
- Continents: `CONTINENT_DEFS` (`wcx` longitude-column, `cy` latitude-row → sets biome, `cr` chunk radius). Add/move/resize here.
- Time: `timeOfDay.speed` (1/840 ≈ 14 min), `dayFactor()` `pow(raw,0.5)` bias.

---

## 6. Pending backlog (what to do next)

> **V9 PROGRESS — Batch 1 SHIPPED (island rework, verified, console clean):** islands are
> now **bigger (r 70–300, was 42–190)** with **organic lobed coastlines** (2–4 sin-lobes +
> fbm on a still-star-convex radial polygon — all collision/docking unchanged via
> `islandRadiusAt`; `landAt` early-out widened 1.2→1.35 for the 1.25 max wobble),
> **terraced elevation** (`isl.tiers[]` = stacked shrunken coast polygons w/ cliff walls;
> 1–3 tiers by size, heights capped; trees carry `tr.z` per-tier; continents/old code
> fall back to single tier), **rocky peak crags** (`isl.peak`, 40% of r>140 islands),
> **pale shallow-water shelf** rings drawn at 1.24×/1.48× (this becomes the shoal damage
> zone in Batch 2), and **satellite islets/sea-stack rocks** (`rock:true` mini-islands on
> the shelf ring — they reuse ALL island collision/render/minimap machinery for free).
> Fishing grounds now also avoid island overlap. Verified: docking, land ejection, fresh
> start, tiered green + ice islands, peak crag. Old saves: world layout changed → save
> was reset (established V8 pattern).
>
> **V9 Batch 2 SHIPPED — reefs, shoals & sandbars (verified, console clean):** chunks now
> carry `reefs[]` (free-standing shallow patches; tropic 30% / desert 24% / else 12% per
> chunk; ~30% are **sandbars** w/ a dry-sand sliver; avoid continents+islands; `EMPTY_CHUNK`
> updated). **`shoalAt(x,y)`** returns the reef object or `{kind:'shelf'}` for island
> shelf water (island coast×1.30, continent ×1.08; rocks excluded). `stepShip` caches it
> as **`ship.shoal`** once/step and slows thrust (reef ×0.78, sandbar ×0.45); `stepDanger`
> applies **speed-gated scrape damage only above speed 55** — `(speed−55)×0.05/s`
> (sandbar ×1.5) + debris fx + shake, so easing into a dock is safe (verified: fast cross
> hit+damage, 15%-sail cross = 0 dmg — beware **water-gate regions confound damage tests**).
> Flashing HUD warning ('shoal water — ease your sail' / 'sandbar!') stacks under the gate
> banner. Reefs chart as dashed pale/sandy circles on the local chart **once explored**
> (`markExplored` now also records reef keys); continents' visual shelf shrunk to 1.06/1.12
> to match their damage zone. NPCs/enemies ignore shoals for now (follow-up).
>
> **V9 Batch 3 SHIPPED — physical roaming storms (verified, console clean):** storms are
> now **moving world cells you can see and outrun**, not a global weather phase.
> `storms[]` = `{x,y,r 480–900,vx,vy (~wind-aligned ~38u/s vs ship cruise 172),life 220–420s,seed}`;
> `stepStorms` (called from `stepWeather`) keeps ≤2 cells alive, spawning 4.2–6.6k out
> every 50–120s (chance scaled by region roughness), drifting, dissipating, culled >12k.
> **Key design: `weather.state` is now DERIVED** — `weather.base` runs the old ambient
> rotation (`WEATHERS` no longer contains 'storm') and `weather.state = stormAt(ship)?
> 'storm' : base`, so EVERY legacy consumer (hull stress, gusts, roughness, rain drops,
> lightning, fog, envGray(+0.55 storm), enemy weighting, market events, repair pause)
> works unchanged and turns on exactly inside a cell. Render: `drawStorms()` after
> `drawWater` (radial dark disc squashed by ISO_Y + 44 seeded falling rain shafts, view-
> culled); squalls chart LIVE on the local chart (dark circle + drift arrow, no explored
> gating). Weather panel gained a **squall-watch line** (bearing/dist/closing-or-passing —
> `closing = dot(stormVel, ship−storm) > 0`, i.e. `(vx*dx2+vy*dy2)<0` with dx2=storm−ship;
> **the sign was inverted in v1 and fixed** — beware). PanelH 82→104 (+`hudTopAvoid`).
> Approach banner '⚠ squall bearing down' (<900u & closing) / '⚠ squall! — reef sail'
> inside. Debug: `__HS.storms`, `__HS.spawnStorm(x,y)`; `setWeather('storm')`/`storm()`
> now spawn a cell on the ship. Verified: outside view + readout, inside (state flips,
> 11dmg/2s full sail, rain/dark/banner), **outrun escape works**, fresh start clear.
>
> **V9 Batch 4 SHIPPED — icebergs, riptides, whirlpools (verified, console clean). V9 CORE
> COMPLETE.** (1) **Flow hazards** (chunk POIs): `flows[]` in genChunk — one per chunk at
> h<0.11 = **riptide** `{r 130–230, dir}` or h>0.958 = rare **whirlpool** `{r 90–150}`;
> avoid continents/islands/reefs; `flowAt(x,y)` scans 3×3. `stepShip` caches `ship.flow`
> and applies forces AFTER thrust: rip = 150u/s² shove along `dir`; whirl = inward pull
> `200*(1−d/r)+40` + 0.7× tangential swirl. `stepDanger`: whirl **core** (<0.3r) grinds
> 6hp/s + shake. Verified: rip drifts an unsailed ship ~69u/2s; whirl pulls rim→eye (minD
> 3u) and damages; **full-sail escape works**. Render `drawFlow` in the flat water pass:
> rip = animated streak segments along the lane; whirl = dark eye + 3 rotating spiral arms.
> Chart: once-explored marks (rip arrow / whirl double-arc; `markExplored` records flow keys).
> (2) **Icebergs** (storm-style local sim, NOT chunk data): `bergs[]` — spawn every 5–12s
> (cap 7, within 0.9–4.1k) **only while `curRegion==='ice'`** and the spawn point is also
> ice/open water; drift 5–14u/s; culled >6.5k or on land. **Collision in stepShip** (after
> the landAt block): pushes out, kills 75% velocity, and `damagePlayer(min(18,(speed−45)*0.09))`
> above speed 45 + shake + SFX. Drawn via the y-sorted drawables list (`kind:'berg'`,
> `drawBerg` = pale faceted floe + crown + foam halo); charted live as white dots.
> Debug: `__HS.bergs`, `__HS.spawnBerg(x,y)`. NOTE: berg damage tests in Frostreach are
> confounded by the ice water-gate drain (as with reefs). NPCs/enemies ignore flows/bergs/
> shoals (one shared follow-up). **V9 remaining nice-to-haves:** chart polygons for island
> shapes; NPC hazard parity. **Next phase: V10 Gunnery & Spoils** (ammo types, utility
> shots, loot drops, inventory, boat naming) per EXPANSION_PLAN.md.
>
> **V10 Batch 1 SHIPPED — floating loot drops + boat naming (verified, console clean):**
> (1) **Loot drops in the water:** `killEnemy`/`killNpc` no longer auto-award — they call
> **`lootSpill(x,y,purse,goods)`** which scatters coin pouches (purse split 1–3) + crates/
> barrels (goods 2-per-crate) at the sink point (`loot[]`, cap none, life 90s). `stepLoot`
> (in the enemies/shots update gate): items bob, drift on the wind (`wind*5`), stop at land,
> **magnet to the ship <110u**, collect <30u → `earn`/`addCargo` + **rising world-anchored
> pickup labels** (`lootFx[]`, `drawLootFx()` full-res after `flushSprites`; avoids toast
> spam). If the hold is full the crate **stays floating** ('hold full!' label); **NOTE
> `addCargo` returns NEGATIVE when over-capacity** (e.g. after hull downgrade) — collection
> clamps it (`Math.max(0,addCargo(...))`), a latent-bug class to watch elsewhere. Loot blinks
> its last 10s, charts as small gold dots on the live local chart, renders in the flat-water
> pass (`drawLoot`: coin glint / crate / barrel for rum+oil, soft gold pulse). Kill toasts
> now say '— spoils adrift'. Boss heart/quest logic unchanged (heart still auto). Debug:
> `__HS.loot`, `__HS.spawnLoot`, `__HS.lootSpill(x,y,coin,{id:qty})`.
> (2) **Name your boat:** `player.boatName` (saved; old saves get a quiet deterministic
> default). Canvas naming dialog `ui.naming` (`openNaming(first)` / `christen()` /
> `drawNaming()`, keydown intercepted BEFORE `keys[k]=true` so typing never steers; ship
> holds position via the `stepShipDrift` gate). Opens automatically on **new game** (Esc
> accepts the suggested name) and via a **Rename button** in the Shipyard tab (Esc cancels).
> Shown: port header (right column, '⚓ the Name'), Shipyard HULL line, Logbook SHIP line,
> death overlay ('THE NAME WAS LOST') + respawn toasts. Debug: `__HS.nameBoat(n)`.
> **Preview note:** port 8801 was held by a stale server — config **`helmsong2` on 8802**
> exists in launch.json and works (same /tmp dir). Also the preview window can open 1px
> wide — `preview_resize` to explicit 1280×720 before screenshots.
>
> **V10 Batch 2 SHIPPED — inventory / Ship's Hold (verified, console clean):** one shared
> renderer **`drawInvContent(ax,ay,aw,btns)`** drawn two ways: at sea via **I** (`ui.inv`,
> `toggleInv`, `drawInventory()` panel; new bind `inv:'i'` in DEFAULT_KEYS + a settings row
> — settings row pitch trimmed 29→27 so 10 keyboard rows still fit) and in port as a new
> **Hold tab** (tabDefs `['hold','Hold']`; 7 tabs fit fine). Content: **cargo grid** (4×4
> slot sprites, code-drawn per-good icons `drawGoodIcon(id,cx,cy,s)` for all 16 goods,
> ×qty chips, hover shows name·count·base); **price ledger** — `recordLedger(isl)` on
> `openPort` snapshots every good's SELL price; per good keeps the best offer seen,
> refreshes the same port, replaces >6-day-stale news (`player.ledger[id]={p,port,t}`,
> `daysAgo(t)` = elapsed/840); rows dim when you don't hold the good; **ammo rack** —
> `AMMO{}` registry (round/chain/grapple/grape/fire/oil/chum w/ costs+descs),
> `player.ammo{}` counts + `player.ammoSel` ('round' = ∞), `selectAmmo(id)` refuses
> unowned, `drawAmmoIcon`, selected slot glows gold. Action bar gained a 7th **hold**
> slot (crate `actionIcon`, lights while open). ui.inv pauses the sim like the map; Esc/I
> close; map and inv close each other; save/load `ammo/ammoSel/ledger`. Debug:
> `__HS.AMMO`, `__HS.giveAmmo(id,n)`, `__HS.openInv()`. **Firing behaviours + the ammo
> economy are NOT in yet — that's Batch 3** (the rack is fully wired for it).
>
> **V10 Batch 3 SHIPPED — gunnery: ammo behaviours, utility shots, ammo economy (verified,
> console clean). V10 COMPLETE.** `fireCannon()` dispatches on `player.ammoSel`:
> **round/chain/fire** ride the full battery (chain dmg ×0.55, fire ×0.8); **grape** = 7
> pellets, ±0.31rad spread, t 0.5s (short cone), dmg ×0.38 each, reload ×1.3; **grapple** =
> single hook shot; **oil/chum** = `util:true` lob that ignores hulls and `splashUtility()`s
> a patch where it lands. `consumeAmmo` decrements & **auto-falls back to round at 0**
> (toast). Effects via **`applyShotFx(target,kind)`** (shared enemies/npcs): chain →
> `slowT=6` (ship-movement branches multiply speed ×0.45 enemy / ×0.5 npc — monsters
> ignore it, no rigging); fire → `burnT=5` (6hp/s, flame fx, kill→killEnemy/killNpc, and
> **burning hulls ignite oil slicks they cross**); grapple → global `grap={t,timer:3.2}`
> hauls ship & target together (170/120 u/s², stops <70u; rope drawn in `drawShots`;
> stepped in `stepSlicks`). **Slicks** (`slicks[]`, r64, 45s): unlit = dark sheen ellipse
> + ship thrust ×0.72 (`slickAt` in stepShip's gateSlow); lit (fire shot skimming z<26,
> or burning ship) = 8s of 10hp/s to ANYTHING inside incl. the player + flame render.
> **Chum** (`chums[]`, r90, 40s): `groundAt` gains up to +0.5 rich inside (fishing buff),
> and `stepEnemies` spawn lure ×1.7 (monster bait — dual use). **Player debuffs:** enemy
> `enemyFireA` picks kinds by tier — navy (warship/manowar/patrol) 25% chain, pirate
> heavies (dread/galleon/corsair) 16% fire; on hit: `ship.slowT=5` (cap+thrust ×0.6) /
> `ship.burnT=4` (4hp/s, **rain/storm douses ×3 faster**), both tick in `stepDanger`.
> **Economy:** port Hold tab sells **crates of 5** under each slot (`buyAmmo`, cost×5 from
> `AMMO[].cost`); action-bar cannon slot shows a mini icon+count badge when special ammo
> is loaded. Per-kind shot visuals in `drawShots` (fire ember+glow, whirling chain pair,
> grapple hook w/ trailing line, dark oil blob, pink chum, small grape pellets) + new
> `'flame'` fx kind. Debug: `__HS.slicks/chums/setAmmo(id)`; `shots`/`grap`/`splashUtility`
> are top-level (reachable from console). NOTE: enemies don't use grapple/grape/utility
> themselves; NPC/enemy ships still ignore slick slow (same family as the shoal follow-up).
>
> **V11 Trade Winds SHIPPED — 3 batches (verified, console clean). V11 COMPLETE.**
> (1) **Region goods + origin-distance pricing:** 8 exclusive trade goods w/ `origin`
> (amber/calm, silk/temperate, jade/tropic, obsidian/desert, ivory/ice, relics/cursed,
> dye/toxic, ambergris/sargasso) + 3 endemic fish w/ `endemic` (icefin/ice, sunfish/desert,
> eel/sargasso; `rollCatch` adds them ~10% at home, they're NOT in `FISH_IDS`). **`TRADE_IDS`
> is now STAPLES-ONLY** (`!origin`) which keeps contracts, port profiles, and market events
> off the exotics for free; `EXOTIC_IDS` holds the rest. Pricing: `REGION_POS{}` = climate-
> band position 0..1; in `priceFor`, origin goods are cheap at source (bm .75/sm .55) and
> gain `1.4×|Δpos|` premium with distance (≈2.1× at opposite bands) — geography IS the
> price. `portRegion(isl)` lazily caches `regionAt` on the port. Exotics are **only stocked
> at origin-region ports**: `doBuy` gates with a toast, and `marketGoods(isl)` filters the
> market list (staples always; exotics at source or when owned; endemic fish only when
> owned) so 27 defined goods still fit the panel (`drawMarketTab` pitch now divides by the
> LIST length, floor 20; local exclusive marked ◆ gold, far exotics show '—' buy).
> Rich traders sometimes carry a random exotic (killNpc). Hold cargo grid widened 4×4→**5×4**
> (20 items can be 20 distinct goods); 11 new `drawGoodIcon` cases.
> (2) **Fish-school depletion:** `groundStock` Map (`g.key` → `{s,t}`), lazily regenerating
> at `STOCK_REGEN` = full in ~2 game days (1680s). `groundAt` multiplies each ground's rich
> by `0.25+0.75*stock`; every player haul `drainGround(0.06)`; **NPC fishers** drain 0.012
> per 2s while within 180u of a ground (`n.drainT` in stepNpcs). Visible thinning:
> `drawGround` scales foam-ring alpha + bird count (5→1) by stock; context prompt says
> 'waters fished thin' below 0.3. Persisted in the save (`stock` array, last 300, rebuilt
> into the Map on load). Chum still adds its flat rich bonus on top (unaffected by stock).
> Debug: `__HS.groundStock/stockHere()/drainHere(amt)`.
> (3) **Ledger v2 + rumours:** `player.ledger[good]` is now an **array of up to 4 port
> entries** `{p,port,x,y,t,rumor?}` (`ledgerNote` dedupes by port, prunes by score);
> **`ledgerScore(e) = p/(1+dist/9000)`** with **E–W-wrapped distance** (`ledgerDist`);
> `bestOffer(id)` picks the entry worth the sail. Hold-tab ledger shows YOUR cargo first
> (best score order) then the rest of the book; rows read 'price · port ↘ ~8.5k · 2 days
> ago' (`fmtDist`/`bearingArrow`, rumours in violet quotes). `leakRumorPrice(isl)` — 40%
> per dock: finds a port 6–14k away (3 direction probes, chunk scan ±2) and records its
> CURRENT sell price for a random good as a `rumor` entry + a dock-gossip toast.
> v1→v2 migration: loadGame drops non-array ledger values (old coordless entries).
> NOTE: rumour prices are snapshots — drift means they're approximate by arrival (fine,
> flavour-accurate). Ledger caps at 4 ports/good; prune uses current-position score.
>
> **V12 Banners of the Sea SHIPPED — 3 batches (verified, console clean). V12 COMPLETE.
> No save reset needed** (rep/war/fort state default-migrates; fort placement derives
> from existing world hashes).
> (1) **Faction core + territory + reputation:** `FACTIONS{}` — crown (navy blue),
> guild (gold merchants), free (Shorefolk green), redsails (pirate red), tide (Tidebound
> cult violet); `FACTION_ORDER`. `portFaction(isl)` lazily assigns port owners by seed
> hash (42% crown / 30% guild / 28% free); `facOfEnemy` (ships→redsails, ghost+monsters→
> tide) / `facOfNpc` (navy→crown, fisher→free, merchant/trader→guild) computed on demand
> AND stamped at spawn (`e.fac`/`n.fac`). **`player.rep{}`** −100..100 (saved):
> pirate kills +crown/+guild/−redsails, ghost/monster kills −tide, piracy −victim's
> faction/+redsails; **prices respond** (`facBuyMul`/`facSellMul` inside priceFor —
> ±5–16%). Shown: port header '⚑ faction · standing' beside the title (NOT at py+112 —
> that row is the tabs), Harbour tab 'STANDING WITH THE POWERS' chip row, chart port
> dots + worldChart dots in faction colours, region-strip ⚑ + 'Entering X waters' toast
> (`curWaters` throttled 0.5s in computeEnv, toasts only on faction CHANGE). Merchant/
> trader flags now guild-gold. Debug: `__HS.FACTIONS/rep()/setRep(f,v)/waters()`.
> (2) **NPC-vs-NPC combat + war:** damage is **source-aware** — `hitEnemy/hitNpc/killEnemy/
> killNpc` take `src` ('player' default); non-player kills spill 70% loot with a
> witnessed-only toast and NO player stats/notoriety/rep; SFX.hit gated <800u. Shots
> carry `{fac, vsPlayer, src}`; stepShots: non-friendly shots only hurt the player when
> `vsPlayer`, and hit any RIVAL-faction enemy/npc (skip shooter). **Pirate quarry pick**
> (ships/deepmaw/serpent, every 1.5s): nearest rival npc within 800 beats the player at
> >85% of the player's distance (war ships always prefer fleets); movement/fire use
> tdist/tang; serpent strike + jelly pulse damage npcs too (kraken/angler stay player-
> only). **Navy engagement**: peaceful navy npcs scan 560 (war: unlimited) and run down
> pirate SHIPS, firing with **aim lead** (`lead=d/300`, aim at pos+vel×lead — without
> lead, circling duels never resolve; both NPC fire sites lead now). **War zones**
> (`war`/`stepWar`, transient): every 5–10 min a 55% roll places a 1900u contested circle
> 3.2–8k from the player (Crown vs Red Sails), 7–11 min duration, toast with bearing;
> while player <4.2k, skirmish pairs spawn every 7–15s (cap 6 `war:true` ships, brig/
> corsair vs patrol, pre-targeted); war ships get farT 60s (not 6/8) so fights persist
> beyond the horizon. Chart: dashed red+blue ring + ⚔ + 'contested waters'; guide arrow
> <6k. Debug: `__HS.warNow()/startWar(x,y)/endWar()`.
> (3) **Forts + garrisons + cult stronghold:** `fortOf(isl)` lazy-deterministic — big
> (r≥150) non-rock non-continent islands: crown/guild PORT islands 40% get a fort
> (max 380hp); unported islands in danger≥1 water 35% → **redsails hangout**; cursed
> region 60% → **tide stronghold** (500hp, drops relics+pearls). `fortPos` sits it at
> 0.42r from centre; `fortState` Map (persisted `forts` in save) with hp/aggro/rebuild —
> **razed forts rebuild after ~3 game days**. `stepForts` (±1 chunk, fills `nearForts`
> for stepShots): **shore guns** fire 13dmg lead shots <640u at hostiles
> (`fortHostile`: redsails/tide always, crown at bounty>50, others rep<−60, or aggro);
> **garrison sorties** every 22s while aggroed (≤2 defenders: brig/corsair, ghost, or
> hostile patrol, tagged `fortKey`). Player shots vs forts resolve via `nearForts`
> (<34u, z<46): first blood = −10 rep + toast + 45s aggro; destruction = `fortFalls`
> (bursts, loot spilled OFFSHORE toward the player at isl.r×1.35+40 — never on land,
> −30 rep owner, +10 to their enemies). `drawFort` = code-drawn iso keep (curtain walls,
> crenellations, tower, faction banner, gun ports, siege hp bar; rubble+smoke when
> razed) drawn with the island in the y-sorted pass; chart shows a faction-colour square
> (dim when razed) once explored. Siege is DEADLY at point-blank (guns+sortie sank the
> test sloop in ~15s — intended, danger is sought out). Debug: `__HS.fortState/
> nearestFort()/damageFort(n)`.
> **V12 follow-ups:** faction relations don't drift over time (static war rolls only);
> no faction-vs-faction fort captures yet (flip mechanic deferred to V14 alongside
> player capture); guild has no distinct warship (sorties use patrols).
> **Testing gotcha:** the preview tab runs rAF LIVE between preview_eval calls — state
> (aggro timers, drift) changes between evals; run multi-step scenarios inside ONE eval.
>
> **V13 Many Flags — FIRST SLICE SHIPPED (3 batches, verified, console clean; world
> regenerates port names → saves were reset).** The culture SYSTEM + 3 kits are live;
> the remaining kits from the plan (Mayan/Greek/Egyptian/Roman/…) are content drops on
> this scaffold. Cultures are WHERE (climate bands + hemisphere); factions stay WHO.
> (1) **Culture system:** `CULTURES{}` — euro 'the Midlands' (baseline), norse
> 'the Northmen' (cold, t<0.24), east 'the Dawnlands' (mid-band eastern hemisphere,
> lon≥0.52), desert 'the Sunborn' (equatorial, t>0.72); `cultureAt(x,y)` (same latitude
> math as regionAt), `portCulture(isl)` lazy-cached. **Port names** per culture via
> `CNAMES` part-tables inside `portName` (Thorvik/Kald Froststad · Meishima/Sorahama ·
> El Sirair/Qassur; euro keeps NAME_A/B/C) — NOTE portName/portProfile derive culture
> from (cx·CHUNK) centre, keep signatures. **Captains** carry home names (`CAPTAINS_C`
> + `captainName(x,y)` used in spawnNpc). **Markets favour home produce** — `CULT_PRODUCE`
> (norse furs/timber/iron · east tea/cloth/pearls · desert spice/salt/cloth), 60% bias
> per produce pick in portProfile (verified: norse band tally timber 62/iron 56/furs 41).
> Port header line reads 'a harbour of the Midlands · region · lore'.
> (2) **Architecture:** `drawPortHouse(isl)` called from drawPier — every port now has a
> shore building at the pier root (30u inland, proj-drawn, col()-tinted): euro stone
> cottage (red gable, chimney, lit window), norse stave hall (dark planks, crossed gable
> beams, rune stone w/ cyan glyphs), east two-tier pagoda (upturned eaves, gold finial),
> desert adobe dome + gilt-tipped obelisk. Pier pennants fly `CULTURES[].pennant`.
> (3) **Culture hulls:** three new VESSELS + three new RIG kinds in drawVesselRig —
> `longship` (rig 'striped': one broad square sail drawn as 4 alternating cream/red
> stripe quads; body flags `shields` = disc row on both rails + `dragonProw` = carved
> quadratic neck+head at the stem), `junk` (rig 'batten': trapezoid sails, foot narrowed
> ×0.68, 3 batten lines interpolated between edges; high sterncastle), `dhow` (rig
> 'lateen': one triangle A/B/C hung on a raking yard stroke; mast has no .sail key).
> **Civilian npcs only** (merchant/trader/fisher) get `n.look` + culture sail tint at
> spawn by `cultureAt` — `drawEnemyShip` renders `vcfg(e.look || e.type)`; navy/ghosts/
> pirates keep faction hulls. Fishers in culture waters lose the netboom visual (their
> culture cfg lacks the flag) — acceptable, noted.
> **V13 deferred:** remaining 5–6 culture kits (pure content on this scaffold: CNAMES
> entry + drawPortHouse branch + VESSELS hull + cultureAt band); per-culture cult
> monsters; per-culture music flavour; island flora per culture.
>
> **V14 Your Flag SHIPPED — 3 batches (verified, console clean). THE V9→V14 EXPANSION
> PLAN IS COMPLETE.** No save reset needed (fleet/hold/banner default-migrate).
> (1) **Fleet escorts:** `player.fleet` (≤2, `{cls,name,hp}`, persisted) ↔ live
> `escorts[]` hulls (`syncEscorts` on load/buy/loss). `stepEscorts`: formation stations
> off the player's quarters (±0.55 rad, 72u; catch-up speed `min(d·1.3, shipSpd·1.35+40)`
> — converges to a ~130u trailing convoy), `keepOffLand`, and **auto-gunnery** (nearest
> enemy <380, led shots, 16dmg/2.2s). Escort shots are `escort:true` — the friendly
> branch SKIPS npcs and forts for them (no accidental piracy/sieges); escort kills
> credit the player (they're your guns). Enemy `vsPlayer` shots hit escorts within 20u
> (`damageEscort` — they screen you); at 0 hp the ship is lost from the fleet (toast).
> Docking repairs the fleet. UI: **Crew tab** 'YOUR FLEET' section (rows + hp bars +
> Dismiss; buy row Sloop 500/Cutter 800/Merchantman 1200/Frigate 1600, hidden at cap).
> Rendered via `drawEscort` → pseudo-object into `drawEnemyShip` with the player's
> cosmetic hull/sail colours + your-banner flag; y-sorted (`kind:'esc'`). Debug:
> `__HS.escorts/fleet()/giveEscort(cls)`.
> (2) **Your stronghold:** `FACTIONS.you` added (excluded from `FACTION_ORDER` so the
> standing row doesn't list yourself). Sail to any **unclaimed razed fort** (<260u) →
> Space → `claimRuin` (800c): `fortState` entry gets `owner:'player'` (persisted as a
> 4th tuple field), hp restored, `player.hold={key,x,y}`. `fortFac(f,st)` maps owned
> forts to 'you' (banner/chart colours); `fortHostile` never fires on you; in stepForts
> **your guns auto-engage enemies** <640u (led 13dmg shots, `fac:'you'`, vsPlayer:false
> — can stray into npcs: 'lost to your guns' flavour, no blame). Space at your hold →
> **stronghold panel** (`ui.hold`/`drawHold`/`holdButtons`, gated like ui.inv): free
> repair on entry, **warehouse vault** `player.vault` (cap 80, `storeGood`/`takeGood`
> per-good Store/Take buttons). `homeWaters` reports 'you' within 1700 of the hold
> ('Entering your waters — the banner holds'); guide arrow '⌂ your hold' 0.4–7k;
> open chart labels the fort 'your hold'. Debug: `__HS.hold()/vault()`.
> (3) **Found your faction + diplomacy:** the naming dialog is generalized —
> `ui.naming.kind==='faction'` branch in `christen()` sets `player.banner={name,col}`
> and rewrites `FACTIONS.you.name/adj`; opened from the stronghold panel ('Raise your
> banner' / 'Rename banner') beside a **6-swatch colour picker** (sets FACTIONS.you.col
> live — escorts' flags, fort banner, chart squares all follow). Founding fires
> rep-dependent reaction toasts (guild congratulates ≥20, crown suspicious <0, redsails
> toast). Banner persisted + re-applied on load. **Diplomacy (Harbour tab, under the
> standing chips):** *Red Sails tribute* — 250c → `player.truce = elapsed+1680` (2 days);
> in stepEnemies, redsails ships with no npc quarry and not at war get `standoff` —
> they shadow at ≥380u and never fire (verified: 15s beside a brig, 0 damage). *Guild
> trade-pact* — at guild ports with rep≥20, 400c → `player.pact` → `facSellMul` ×1.08 at
> guild harbours; **revoked inside `repAdj`** the moment guild rep goes below 0 ('tears
> up your pact'). Both persisted.
> **V14 deferred (post-plan backlog):** automated trade routes for fleet ships (passive
> income + raidable); sieges AGAINST your hold / faction fort captures (the 'capture is
> a setback' loop); escort ship-class variety beyond the 4 player classes; win-condition
> flavours. NOTE: `nearRuin` requires `!player.hold` — one stronghold at a time by design.
>
> **FULL PLAYTEST + FIX BATCH (see docs/PLAYTEST_V14.md for the whole report):** played
> every system V1→V14 with real inputs; console clean; perf 0.12ms/tick under worst-case
> load. 7 issues found and FIXED same-session: (B1) fishing ground drain 0.06→0.035/catch
> (was stripping a rich ground in one session); (B2) `respawn()` now calls `syncEscorts()`
> at its END (after the ship moves — first attempt placed it before the teleport and the
> fleet stayed stranded) + escort idle catch-up cap 60→110 u/s; (B3) harbour 'all taken'
> now advances `ry` (was overlapping MARKET NEWS); (B4) harbour pitches tightened
> (contracts 24 / offers 28 / news 22) + news clipped at `footY-30` with '…'; (P1)
> chain-slowed ships draw dark limp sails + no flag (`e.slowT>0` in drawEnemyShip — npcs
> included, escorts' pseudo-objects unaffected); (P2) stronghold warehouse columns show
> '… and N more' past 13 rows; (P3) fort first-blood grace — `st.fireT=max(fireT,1.5)`
> when aggro is freshly set, so the warning toast lands before the first ball.
>
> **NEW-PLAYER PLAYTEST + FIX BATCH (docs/PLAYTEST_NEWPLAYER.md):** a genuine fresh-eyes
> first session (factory reset, screen-taught only). 8 findings, ALL FIXED same-session:
> (NP-1, the big one) **burn-death attribution** — `applyShotFx(t,kind,src)` now stores
> `t.burnSrc`; both burn-tick deaths use it (`killNpc(n, n.burnSrc||'redsails')` /
> `killEnemy(e, e.burnSrc||'player')`) — previously a PIRATE's fire shot burning a
> merchant made the PLAYER wanted (repro'd organically: WANTED 40 without firing a shot);
> (NP-2) Open Sea calmed — spawn clock (11+r·11)→(18+r·16), temperate danger 1.0→0.8,
> dread/galleon gates ≥1.0→≥0.8 to preserve their spawn regions (verified: 5 ambushes
> per 2.5 min → 1); (NP-3) contracts carry `fac` (set in portContracts) + ✕ **Abandon**
> button per YOUR CONTRACTS row (`abandonContract`, −2 rep with issuer) — previously
> unfundable deliveries bricked all 3 slots forever; (NP-4) the active **Deep Song step
> leads the top-right tracker** ('❖ …', violet; hudTopAvoid accounts for it) + christen()
> fires a 'logbook marks a first task' toast on first games; (NP-5) intro key list gained
> I + Esc rows; (NP-6/7) fishing prompt: 'Space to stow the net · slower is better'
> above speed 45 (NOT 70 — net drag keeps trawling under ~70, found in verification);
> (NP-8) Logbook row renamed 'Monsters slain'.
>
> **SENIOR-DEV QA PASS + FIX BATCH (docs/QA_SENIOR.md):** regression/edge battery —
> Deep Song end-to-end, Tidebound fort, wrap seam, poles, 8-sim-min soak (all pools
> bounded, ~60× realtime), UI-toggle storm, awkward-moment saves. 4 defects, ALL FIXED:
> (SD-1, CRITICAL) **quest Leviathan never spawned** — spawner required `quest.step===2`
> inside cursed but arrival advances 2→3 same frame; now `===3` (index.html ~:4207;
> kill handler at :4129 always expected 3). Verified: boss rises ~9s after entry.
> (SD-2) **stronghold now wrap-aware** — `wrapDX/holdDist/holdNearestX` helpers used in
> homeWaters/handleAction/prompt/drawGuides (guide targets nearest period copy, like
> waypoints); circumnavigation verified east & west. (SD-5) autosave skips while
> `ship.sinking` + loadGame clamps hp to ≥20% maxhp (old hp:0 saves load 'barely
> afloat'). (SD-4) **short-window UI scale enabled** — `computeUIScale: US =
> clamp(uiScaleMul·min(1, H/740), 0.7, 4)`; the dormant US design-space infra now does
> its job (18-row market fits at 900×600; NOTE at 720p US=0.973, ≥740px = native).
> Plus hardening: `frame()` self-heals missed resize events (stale canvas seen under
> viewport emulation; also covers mobile orientation). Withdrawn: 'ice gate no drain'
> was a harness artifact (port left open — drains pause while docked; `dockAt()` in
> tests must be paired with `closePort()`).
>
> **UX/UI DESIGN REVIEW + FIX BATCH (docs/UX_REVIEW.md):** every screen audited; 9
> findings ALL FIXED: (UX-1, the big one) **the action bar is now a real control
> surface** — `ACTION_SLOTS[].click` handlers, `actionBarRect`/`actionSlotRects`
> refreshed in drawActionBar and hit-tested in mousedown BEFORE fireCannon (slots act:
> net→handleAction, lantern/chart/hold/mute→toggles; cannon & anchor slots + bar
> dead-space swallow; gold hover ring on clickables) — previously clicking any slot
> FIRED THE CANNON (misclick piracy risk). (UX-2) UI-scale design height 740→**700**
> so 1280×720 renders native-crisp (US=1) — the 740 value from SD-4 made 720p blurry;
> required (UX-5) shipyard trim: upgrade pitch 30→29 + crest chips fnt(10)/3-char so
> the tab clears the footer at a 696px panel. (UX-3) chart legend rebuilt — all 7
> region swatches + 'ports & forts fly their owner's colours' (old legend was missing
> V8 regions and showed a pre-faction Port swatch). (UX-4) Logbook 'YOUR FLAG' section
> (banner · stronghold · fleet · standing chips; achievements two-columned for space).
> (UX-6) crew-tab hull bar labelled+backed beside Dismiss, 'hire another escort:'
> microcopy. (UX-7) diplomacy buttons verb-first ('Pay 250c'/'Sign · 400c').
> (UX-8) fortress = 'stronghold' in ALL player-facing text; 'Hold' = cargo only.
> (UX-9) journey rows show 'begins: <first step>'. Bonus: catch-toasts hidden while
> the chart is open (edges peeked out past the panel).
>
> **USER-DIRECTED MENU REWORK (user feedback w/ reference image):** (1) **Hold-tab
> overlap FIXED at the root** — the ammo rack (~400px) crossed the ledger column
> (ax+348); the ledger is GONE from `drawInvContent` (Hold = cargo grid + ammo only)
> and lives on its own full-width tab (`drawLedgerTab`, used by BOTH the port and the
> captain's screens). (2) **The I-screen is now the tabbed 'captain's screens'**,
> available ANYWHERE incl. docked (drawn over the port; Esc closes it first — Esc
> chain reordered inv→port→map→hold→pause): tabs **Hold · Ledger · Factions · Log ·
> Lore** (`ui.invTab`). Factions tab = each power's flag/desc/whereabouts + standing
> (FACTIONS registry gained `desc`/`home` text; shows YOUR banner once raised).
> Old Logbook split: **Log** = stats/ship/your-flag; **Lore** = achievements (2-col) +
> sea lore (last 8). (3) **Port slimmed to 6 tabs** — Market · Ledger · Shipyard ·
> Crew · Harbour · Quests (Hold & Logbook removed; ammo crates still purchasable via
> the I-screen Hold tab while docked — `inPort = !!ui.port`). (4) **Always-on-top
> status widget** (`statusPill`/`drawStatusWidget`, drawn LAST in the UI pass):
> medallion-pill style per the user's reference — coin · hold x/y · '⚓ the <boat>' at
> top-left, above every panel. Redundant readouts REMOVED from the port header, the
> I-screen header, the stronghold panel, and the old top-left coin strip; region/
> WANTED/heart strips shifted down (y122/158/194) and `hudTopAvoid` updated.
>
> **USER-REPORTED FIX — ship-name collision (the 'wrong name' flotsam toast):** the
> LORE fragment 'Wreck of the Meridian' shared a name with the SHIPNAMES pool that
> suggests the PLAYER's boat name — after renaming, picking up that flotsam read like
> the game announcing the boat's old name. Fixed three ways: the fragment is now
> 'Wreck of the Amberline' (a name outside every pool; lore saves store INDEXES so old
> saves are fine); the toast is self-describing ('Flotsam recovered — sea lore: "…"');
> and a new **`pickShipName()`** (filters out the player's current boatName,
> case-insensitive) is used everywhere strangers get named — npc captains' shipName,
> buyEscort, __HS.giveEscort — so no NPC or escort ever sails under your flagship's
> name. NOTE: keep future LORE/RUMOR ship references OUT of the SHIPNAMES pool.
>
> **PLAY-FEEL PASS ON THE REWORKED MENUS (2026-07-02 — fresh-eyes run of the
> new-player flow; 5 findings, ALL FIXED same-session, verified, console clean):**
> (PF-1) the status widget read '⚓ the' before christening (boatName is '' until
> `christen()`) — `drawStatusWidget` now shows 'unchristened' until a name is set.
> (PF-2, the real one) **the Ledger silently truncated at `rows.slice(0,17)`** —
> `recordLedger` snapshots ALL 27 goods on the first dock, so 10 goods were
> permanently invisible with no indicator; `drawLedgerTab` now takes a `maxY` arg
> (I-screen passes `py+ph-64`, port passes `py+ph-70`), fits rows to the real panel
> height, and ends with '… and N more in the book — goods aboard rise to the top'
> (owned cargo still sorts first, so nothing you carry can hide).
> (PF-3) the context prompt ('Space to cast net') and the catch toast drew through
> the pause menu, colliding with its 'Esc to resume' microcopy — both now gate on
> `!ui.paused`. (PF-4) the intro key list still said 'I — ship's hold & ammo' →
> now 'captain's screens (hold · ledger · log)'. (PF-5) the contracts-full toast
> said 'Logbook full' (a tab that no longer exists) → 'You already hold 3 contracts
> — finish or abandon one'.
> **Everything else in the menu rework held up under play:** all 5 captain's tabs
> at sea AND docked (drawn over the port), the Esc chain inv→port→pause, real-click
> ammo-crate purchase while docked, the action-bar hold slot, M and I closing each
> other, the quest tracker + Quests-tab checklist, live status-widget updates, and
> save/load across reload. (Side note: the tester's sloop sank in ~30s parked in
> Frostreach gate water with the net down — the banner warns, the respawn penalty
> applied cleanly; intended tuning, left alone.)
> **Harness notes:** `preview_click` on `#startBtn` silently no-ops in this setup —
> dispatch `element.click()` via `preview_eval` instead; the stale-capture-surface
> screenshot quirk persists across reloads (page-side dims are correct; verify via
> `preview_eval` reads + readable-if-small screenshots).
>
> **USER-FEEDBACK BATCH — HUD order, boss leash/respawn, chart filters + chart
> art overhaul (2026-07-02, all verified, console clean):**
> (1) **Region strip moved ABOVE the coin/hold/name pills** — it now lives in
> `drawStatusWidget` (always-on-top) at y12; pills shifted to 50/86/122;
> `hudTopAvoid` leftBot 152→158. WANTED/heart strips unchanged.
> (2) **Leviathan leash + respawn rest** (user: boss chased them out of the Cursed
> Deep for minutes and respawned on every re-entry). `stepBoss` now leashes: if
> `regionAt(ship)!=='cursed'` the boss stops attacking (tentacles cleared), swims
> away, and despawns after 5s ('The Leviathan sounds — the deep does not give
> chase'; `e.dead=true`, NO killEnemy → no spoils) setting **`player.bossCd`**
> (persisted in the save next to truce): +90s while quest step 3, +420s post-quest,
> **+840s (a full game day) after a kill**. The spawner (stepEnemies) no longer
> rolls `Math.random()<0.0009`/tick — both quest and post-quest paths use the
> deterministic `bossT` countdown (8s quest / 30–60s post-quest, re-armed on
> leaving cursed) **gated on `world.elapsed >= player.bossCd`**. Verified via a
> scripted eval: spawn → flee → 5s leash despawn + 420s cd → re-entry during cd
> spawns nothing → cd cleared spawns → kill sets 840s.
> (3) **Chart filters** — `mapFilter {factions,wars,goods,bosses}` + toggle chips
> drawn under the open-chart header (`mapBtnRects`, hit-tested at the TOP of
> `mapClick` so chips never drop waypoints). Overlays on BOTH world & local charts:
> **Factions** = `facChart()` cached canvas — port factions are hash-random, so it
> majority-votes ports over a ±2-cell window (40×32 grid) and tints by dominance,
> premultiplied-blurred → soft territory zones; + live fort squares (world) + soft
> harbour-reach glows (local) + a faction legend row. **Wars** = ring/⚔/label at
> the live `war` zone (world+local) or a 'no contested waters at present' note.
> **Goods** = `mapPois()` one-time 120×96 `regionAt` scan; each origin good/endemic
> fish labels its region's densest cell ('◆ Jade', '≋ Icefin'); local ports in an
> origin region get a gold ◆ on their name. **Bosses** = '⚠ Leviathan waters' at
> the cursed centroid (+ live pulsing red dot when risen, in local), tide-fort
> squares w/ label on the NEAREST stronghold only.
> (4) **Chart readability overhaul** (user: 'everything looks like a circle,
> zoomed out it's overlapping colour and shapes'): new **`blurRGBA()`** helper
> (premultiplied 4-ch box blur, optional x-wrap — plain blur dragged zone borders
> toward black) now drives the world chart's climate tint, the open local chart's
> region tint (36×36 field, cached small canvas, built per frame — cheap), and
> facChart → all soft watercolor zones instead of hard 4px patchwork.
> **Islands draw their real lobed coastlines** (`isl.pts` polygon) on the local
> chart when >5px (dot below), continents likewise on the world chart (+ subtle
> coast stroke); world-chart port dots shrunk 1.0→0.8px & alpha .75→.55.
> **Port-label decluttering**: `labelFits` collision test — overlapping labels drop
> (dots stay), and labels stay out of the header/chips + legend bands and off the
> chart edges. **Dark backing strips** behind the open chart's header row and
> legend block. **Pulsing gold halo** around the player marker on the open local
> chart (world already had one). Hazard declutter: reefs skip <2px, flows <2.5px.
> NOTE: `_worldChart`/`_facChart`/`_mapPois` are session caches of the FIXED world
> — if world layout ever changes at runtime, null them.
>
> **Chart header restyle (user follow-up — 'center it, match the menu styling'):**
> the open chart's top row is now a **centred** title ('WORLD/LOCAL CHART' in
> `fnt(12)` gold) with the dim hint inline after it, and the filter row is
> **centred brassButton tabs** (`btnW(label,10,8)`, h26, `font:fnt(10)`, gold
> active state — same look as the port tabs), on a taller 64px backing strip.
> `mapBtnRects` mechanism unchanged (chips hit-test first in `mapClick`).
> **Gotcha found:** `brassButton` leaves `textAlign='center'` behind — the legend
> below expects left/top, so the header block resets alignment after drawing tabs
> (the legend scrambled without it). Overlay labels + the wars note clamp below
> the strip (`Math.max(dy+…, my+76)`); stacked goods labels clamp as a GROUP so
> polar POIs don't overprint; `labelFits` top band is now `my+74`.
>
> **BIG CONTENT + CHART BATCH (user list — legend, zoom controls, smooth map
> text, more bosses, more resources, more land; verified, console clean.
> ⚠ WORLD LAYOUT CHANGED — new continents/goods → reset old saves, established
> V8 pattern):**
> (1) **Legend restyled** — bottom strip 66px, three CENTRED rows (`swatchRow`
> helper, fntS(10), 10px swatches): climates · faction swatches (when the
> Factions overlay is up, else the ports-fly-colours note) · 'you are here'.
> (2) **Zoom controls** — title-row right cluster: `[−] [9.0k] [+]`; +/− call
> `mapZoomIn/Out`; the readout is **click-to-type** (`ui.zoomEdit`, keydown
> intercepted BEFORE `keys[k]=true` like the naming dialog; value in thousands,
> Enter applies `clamp(v*1000, MAP_LOCAL_MIN, MAP_LOCAL_MAX)`, Esc cancels;
> cleared in `toggleMap`). Title+hint now centre in the space LEFT of the cluster.
> (3) **On-map text de-pixelated** — all chart labels (port names, continents,
> 'contested waters', stronghold) switched `fnt`→`fntS` per the two-tier rule
> (VT323 pixelates below ~12px); buttons/titles stay VT323.
> (4) **REGION BOSSES** — every hazard sea now has a named terror
> (`REGION_BOSS{}`: ice 'the Rimehold Kraken' · sargasso 'the Kelpfather' ·
> toxic 'the Blightmother' · desert 'the Sunspine Wyrm' · tropic 'the Reefjaw');
> `spawnMiniBoss(rk)` reuses the existing monster AIs with scaled hp/hit-radius +
> a generic **visual scale wrap in `drawEnemy`** (translate-scale-translate about
> the body — any monster reads boss-sized for free). Same lifecycle as the
> Leviathan: rise timer `bossT` (re-armed per sea via `bossArmedFor`; failed
> land-rolls retry in 3s), **leash** (6s outside their home region → despawn, no
> spoils, 420s rest), **kill → 840s rest**, rests persisted in `player.bossCds{}`
> (the Leviathan keeps `player.bossCd`); boss bar shows the name; kills drop
> region-flavoured loot incl. exotics. Test gotcha: region-edge flapping re-arms
> the timer — test from solid open water (`findOpenWater`), and remember desert's
> densest cells sit ON Kharsun.
> (5) **More resources** — 2 new exotics: **Red Coral** (tropic) + **Dune Glass**
> (desert) (GOODS + `drawGoodIcon` cases; everything else — market gating,
> pricing, ledger, trader loot — flows from `origin` automatically; 29 goods now).
> `mapPois()` returns **up to 4 well-spaced points per goods region** (top-dense
> cells, ≥11k apart) and **2 per boss region** — the Goods/Bosses overlays now
> populate the whole chart (32 goods pts, 12 boss pts).
> (6) **More land** — 3 new continents in `CONTINENT_DEFS` (**Vellmark** NE,
> **Kharsun** equatorial west/desert, **Thalvess** SE; 6 total) and **rare GREAT
> isles** (3.5% of islands roll r 340–520 — small continents in their own right).
> (7) **World-chart label dedup** — `wFits` collision list: continents register
> first, bosses draw before goods, `outlined()` drops a label rather than
> overprint; local-chart overlay labels keep out of the header/legend bands.
>
> **REAL MUSIC (user-provided tracks; verified, console clean):** the procedural
> **3-osc chord pad is REMOVED** (user: 'what sound is always playing? remove
> that') — Web Audio now does **ambience only** (sea brown-noise, bandpass wind,
> combat tension drone, SFX). Background music is now the **4 instrumental mp3s
> cycling** via an HTMLAudio element: `MUSIC{tracks[], el, idx, vol:0.38}` +
> `initMusic()` (called beside `initAudio()` in `start()`) + `playMusicTrack()`.
> Random opener each session, then cycles in file order on `ended`; a broken/
> missing file skips ahead after 5s; '♪ Track Name' toast on each change (skipped
> for the very first track — world isn't running yet). **Mute (P) also zeroes
> `MUSIC.el.volume`** (HTMLAudio bypasses the Web Audio master gain); autoplay-
> blocked starts retry inside `resumeAudio()` on the next gesture. Files live in
> **`assets/music/*.mp3`** (web-safe kebab names — Foggy Horizon · The Lonely
> Horizon · The Long Way Home · Windswept Drift; sources in `~/Desktop/Helmsong
> music/`). Mirror `assets/` to /tmp when deploying, as ever. To rebalance:
> `MUSIC.vol`; to add tracks: drop the file in assets/music/ + one line in
> `MUSIC.tracks`.
>
> **COMBAT MUSIC (user-provided; verified, console clean):** a second HTMLAudio
> channel `MUSIC.cEl` crossfades against the calm cycle (`stepMusic(dt)`, called
> at the TOP of `stepAudio` so music runs even if Web Audio init failed; ~0.9s
> fade). **In combat while:** a boss/terror is risen, any hostile (enemy or
> hostile npc) is within 650u, or `bounty>0` with a hostile navy hunter within
> 950u; **7s of clear water stands down** (hysteresis so music doesn't flap).
> Track pick: terrors open on a random **Boss Combat I/II**, skirmishes on
> **The Road to the Last Light** (`MUSIC.combat[]`, cycles within the pool if a
> fight outlasts a track). The fading channel PAUSES at zero volume — so the calm
> track resumes exactly where it left off after a fight. Mute sets both channels'
> volumes immediately (stepMusic doesn't run while the game is paused);
> `resumeAudio()` gesture-retries only the ACTIVE channel. No '♪' toasts for
> combat tracks (would spam every fight). Sources in `~/Desktop/Helmsong music/
> combat music/` → `assets/music/boss-combat-{1,2}.mp3` +
> `the-road-to-the-last-light.mp3`. Tune: `MUSIC.cVol` (0.44), the 650/950u
> radii, and the 7s stand-down in `stepMusic`.
>
> **REGION THEMES (user-provided; verified, console clean):** third HTMLAudio
> channel `MUSIC.rEl` (looping) for waters that own their own music —
> `REGION_MUSIC{}`: **cursed → 'The Cursed Deep'**, **ice → 'The Frozen Void'**
> (`assets/music/the-cursed-deep.mp3` / `the-frozen-void.mp3`; sources were on
> `~/Desktop/`). One priority rule in **`musicActive()`** (used by stepMusic,
> toggleMute, resumeAudio): **combat > region theme > calm cycle**. stepMusic's
> channel loop is now generic — the active channel resumes-if-paused + fades up,
> the others fade down and PAUSE at zero (calm keeps its track position across
> both fights and region visits; a region theme keeps its position across a fight
> in its own waters). Theme announces with a '♪ name' toast on first entry; a
> broken theme file sets `MUSIC.rErr` → falls back to the calm cycle. To add a
> region theme: one `REGION_MUSIC` line + the mp3. Tune `MUSIC.rVol` (0.40).
> Test note: Frostreach/cursed ambient spawns flip the music to combat mid-test —
> `killAll()` each tick when checking pure region-theme behaviour.
>
> **TABBED SETTINGS — Sound / Graphics / Controls (user request; verified,
> console clean):** the O-screen is now 'Settings' with tabs (`ui.setTab`,
> defaults 'sound'; pause-menu button renamed Settings; intro key line updated).
> All options live in **`OPTS{}`** (music, sfx, shake, rain, brightNights,
> pixel), persisted to localStorage **`helmsong_opts`** (separate from save +
> binds), loaded at boot via an IIFE.
> **Sound tab:** two draggable **sliders** (5% steps — track+knob drawn in
> `drawSettings`, rects in `settingsSliders`; mousedown starts `ui.sliderDrag`,
> canvas mousemove drags, window mouseup releases) + a mute row. Volumes apply
> INSTANTLY via **`applyAudioOpts()`** (toggleMute now delegates to it):
> `AUDIO.master = 0.55*OPTS.sfx` (all Web Audio: ambience + SFX + tension) and
> every music channel × `OPTS.music` (stepMusic fade targets multiplied too).
> **Graphics tab:** cycle buttons — **Pixel style** (user simplified from a
> 4-way size cycle): **Auto** (adaptive pixel-art, OPTS.pixel=0) or **No pixel**
> (crisp full resolution, OPTS.pixel=1 → PIXEL=1, buffer at native size —
> ~0.36ms/tick, fine); old saved 2/3/4 values collapse to Auto in loadOpts.
> (`pixelForSpan` returns the override; boot line `PIXEL=pixelForSpan(480)`
> right after OPTS load so the preference holds from frame 1; changing it calls
> `applyZoom(viewSpan)`), **Screen shake** on/off (gate inside
> `addShake`), **Rain & spray** full/light (drop spawn ×0.45, cap 360→160),
> **Bright nights** (night floor `envMul` 0.58→0.74 in computeEnv).
> **Controls tab:** the old rebinding rows unchanged (pitch trimmed 27→26 to fit
> under the tab row; Reset defaults only on this tab). Buttons with `.act` fire
> directly in the settings mousedown branch (before the rebind fallback).
>
> **REAL SFX — BATCH 1 WIRED (user's Suno files; verified, console clean):**
> 34 files in **`assets/sfx/<name>-<i>.mp3`** (sounds 1–16 of
> `docs/SOUND_PROMPTS.md` + bonus `lightning` ×3; sources in `~/Desktop/Sound
> effects/`, copy-normalized). Engine: **`SFX_FILES{}`** (name → variation
> count) → `loadSfx()` (called at the end of initAudio; fetch+`decodeAudioData`
> into **`SFXB{}`** pools) → **`playSfx(name, vol)`** (random variation through
> a per-shot gain into `AUDIO.master` — so mute + the Sound slider + overlap all
> work for free; returns false when a pool is empty). The `SFX{}` registry now
> tries the file first and **falls back to the old procedural blips** — future
> batches are: drop files, bump the count in SFX_FILES (click/net-cast/net-stow/
> loot/lantern-on/lantern-off are pre-listed at 0).
> **New call sites wired:** sail raise/reef + anchor (key-EDGE triggers via
> `ship.sfxUp/sfxDn/sfxAnc` flags in stepShip — not per-frame while held; anchor
> only when speed>12), land bump (speed>60, 1.6s throttle `ship.bumpT`), berg
> impact (now `collision`, was `hit`), **dock** (openPort) + **castoff**
> (closePort), **reef scrape** (1.5s throttle `ship.scrapeT` in stepDanger),
> **cannonball splash** on water-landing misses (<800u, distance-volumed),
> **sink-enemy** on any ship death within 1200u (killEnemy + killNpc, incl.
> NPC-vs-NPC), **sink-player** in startSink, and **distant storm rolls**
> (`AUDIO.rumT` every 9–18s in stepAudio when a squall is within r+3500 —
> `SFX.rumble`, louder inside). `SFX.thunder` (the bolt) now plays the sharp
> `lightning` files; the `thunder` files are the distant rolls. Volumes are
> first-guess (0.45–0.9 per call) — tune by ear in the SFX registry.
> **'No sound' debugging (user report, resolved-by-checklist):** the chain was
> PROVEN working via an AnalyserNode on AUDIO.master (fired cannon → measured
> 0.61 output peak). If effects are silent but music plays, check IN ORDER:
> (1) the **Sound & ambience slider persisted at 0%** (`helmsong_opts`.sfx —
> silences files AND fallback blips AND sea/wind while music continues — the
> exact symptom); (2) stale tab — hard-refresh; (3) P mute. Hardening added:
> `playSfx` resumes a suspended AudioContext, and on fetch failure (file://
> double-click — CORS blocks fetch but NOT media elements) file URLs land in
> **`SFXH{}`** and play via throwaway HTMLAudio clones (volume ×0.62×OPTS.sfx,
> mute respected) — so SFX now work from file:// too.
> **Baked composite SFX (ffmpeg, user recipe):** `chain-shot-1.mp3` = cannon
> boom + chains overlay at +90ms/0.7vol; `grapple-1.mp3` = whoosh → clang
> (+300ms) → rope haul (+650ms). Recipe: per-layer `silenceremove` (head), 
> `adelay`, `amix normalize=0`, `alimiter 0.89`, reverse-trim tail (sources in
> `~/Desktop/sfx-chain-shot/` + `sfx-grapple/`; ffmpeg at `~/ffmpeg-bin/`).
> Wired: `fireCannon` plays `SFX.chainShot()` for chain rounds (falls back to
> plain fire); `SFX.grapple()` fires in applyShotFx when the hook BITES.
> **Fire set (same recipe):** `fire-ignite-1.mp3` = splash → fire-start (+300ms);
> plays at every ignition — slick catches (both the fire-shot skim and a burning
> hull crossing one, gated <700u), your ship catching fire, and applyShotFx fire
> payloads landing (<700u, 0.55vol). `burning-1.mp3` = head/tail-trimmed
> **loop**, run as a persistent looping BufferSource in stepAudio (`AUDIO.burnG`,
> lazily created once the buffer decodes): gain `setTargetAtTime` →
> `0.5*SFX_GAIN` whenever the player ship burns or any lit slick / burning hull
> is within 520u, back to 0 otherwise (verified: gain 0.19 while burning, fades
> after douse).
> **AMBIENT LOOP MANAGER (replaces the bespoke burning block):** `AMB_LOOPS{}` =
> name → live target-gain fn; `stepAmbLoops(now)` (from stepAudio) lazily builds
> one looping BufferSource + gain per entry (random variation pick) and chases
> the target via `setTargetAtTime(target*SFX_GAIN, now, 0.45)`. Entries:
> **burning** (fire near player), **rain-light** (`weather.state==='rain'`),
> **rain-storm** (`==='storm'` — the two are mutually exclusive by state),
> **whirlpool** (proximity swell: nearest whirl rim within 220u → 0→0.55).
> Adding a loop = one SFX_FILES entry + one AMB_LOOPS line. Lantern one-shots
> wired in `toggleLantern` (match strike / puff — note the lantern STARTS lit,
> so a fresh game's first L plays lantern-off). Later same batch-line: **war**
> loop (distant cannon rumble, swells within 2600u of the contested rim, floor
> 0.15 inside) + **port** loop (0.55 docked / 0.3 within 380u of a pier) +
> **gulls** ×3 one-shots on a sparse 10–24s timer (`AUDIO.gullT`) wherever birds
> already wheel — fishing grounds, islands close aboard, harbours (suppressed
> while docked; the port loop carries its own gulls). 75 SFX buffers.
> Still pending: whale (user's folder arrived empty), click, dolphin/sprite/
> ghost, map/book, horn, 3 stingers.
>
> **FINAL SFX BATCH — THE SET IS COMPLETE (2026-07-03; 86 buffers; verified,
> console clean). Only `whale` (empty folder) + `dolphin` (never generated)
> remain — both pre-wired-ready.**
> **UI CLICK everywhere:** `SFX.click()` fires on every button surface — the
> generic `if (hit(b)){ b.action(); }` lists (naming/pause/inv/hold/port + the
> port touch handler), settings buttons, chart header buttons (filters + zoom),
> and action-bar slots. Clicks are click-1/2 capped at 0.5s (`atrim`).
> **map** ×2 parchment on `toggleMap` (open AND close) · **book** ledger-flip on
> `toggleInv` · **horn** on `claimRuin` + banner raise (both were SFX.upgrade) ·
> **sprite** shimmer on sea-sprite sightings (the `questEvent('sight_…')` site) ·
> **ghost** moan on a sparse 14–26s timer (`AUDIO.ghostT`) when any spectral
> hull (enemy or npc) is within 560u.
> **STINGERS (6s Suno pieces, head-trimmed + 1.1s tail fade):**
> `sting-death` plays in startSink alongside the sink groan — and **stepMusic
> hushes ALL music channels while `ship.sinking`** (`hush` factor 0 → the sting
> stands alone; music resumes after respawn). `sting-victory` replaces
> SFX.upgrade on Leviathan AND terror kills (the fast combat-music duck-out
> means it rings over a quieting mix). `sting-quest` replaces SFX.upgrade at
> the Deep Song finale. Volumes 0.75–0.8 with the usual SFX_GAIN on top.
> **Wanted gong + fort cannons:** `wanted-1.mp3` (metal gong, ring tail kept)
> tolls ONCE — `becomePirate` gates on `player.bounty <= 0` before the bounty is
> added, so repeat piracy acts don't re-gong until the bounty is cleared.
> `fort-cannon-1/2.mp3` replace `SFX.fire` at BOTH fort gun sites in stepForts
> (hostile shore guns + your stronghold's auto-guns, distance-volumed <1400u).
> `fort-falls-1.mp3` = 3-layer collapse (user's Sound 1→2→3, each entering
> ~600ms before the last ends: delays 0/1300/2600ms, 3.9s total) — plays in
> `fortFalls()` (was SFX.upgrade, which still serves as its fallback).
> **MONSTER BATCH (18 files, 60 buffers total):** `monster-death` ×4 (killEnemy
> top block — non-ship deaths ≤1200u, bosses at 0.95), roars via **`SFX.roar
> (kind, v)`** — `roar-serpent` ×2 (wind-up telegraph <900u), `roar-kraken`
> (reach state), `roar-deepmaw` ×3 (spew, 5s throttle `e.roarT`), `jelly-zap`
> ×2 (pulse damage, replaced SFX.hit); **spawn roars**: spawnBoss →
> `leviathan-rise` (the 'A_colossal_sea_monst' file the user dropped in the
> hull-alarm folder — clearly the leviathan prompt take, flagged to them),
> spawnMiniBoss → its kind's roar at 0.9. `tentacle` ×3 = leviathan strikes
> (0.7s throttle `e.tenT`) + kraken GRAB (was SFX.hit). `hull-alarm` (wooden
> creak) every 7s while hp<25% (`AUDIO.alarmT` in stepDanger — NOTE: throttle
> baseline means no alarm in a session's first 7s). `net-cast` on cast;
> net-stow wired but file still pending (empty Desktop folder). Angler chomp
> still uses plain `hit` — no angler file yet.
>
> **AMMO BUYING BACK IN PORT (user: 'we lost the ability to buy ammo'):** the
> menu rework had moved crates to the docked I-screen only — technically there,
> practically undiscoverable. `drawMarketTab` now ends with a **POWDER & SHOT
> strip** riding the Set-sail row (user: 'too close to the goods — line it up
> with Set sail'): title at footY−47, tiles footY−33, price buttons at footY+7
> h22 (inside the Set-sail band; Set sail itself is right-aligned, no clash).
> 6 icon tiles (AMMO minus round; `drawAmmoIcon` + owned count) with a `cost×5`
> buy button each (`buyAmmo`). (Budgeting lesson: fnt(13) glyphs are ~23px —
> a 13px 'text height' allowance overlapped rows.)
> **DIFFICULTY PASS (v0.16.0 — user: 'starting health too high, enemy damage
> way too low, bosses need AOE you must dodge'; verified, console clean):**
> Sloop hp **100→70** (other classes untouched); `enemyFireA` base dmg
> **11→15 / heavies 15→20**. **Leviathan:** tentacles per volley `1+phase` →
> `2+phase` (spread 95→110), tentacle dmg 15→22, spew 10→14, and a NEW
> **shockwave** (phase≥2, every `8−phase`s): 1.25s red telegraph ring at 270u
> around the body (drawn in drawBoss as a squashed ellipse), then a burst —
> inside 270u = 24 dmg + 240u/s radial knockback; outside = a splash. Sail OUT
> of the ring. State on the boss: `e.waveT`/`e.wave{t,hit}` (wave logic sits
> AFTER the leash early-return, so it only runs in cursed).
> **All 5 terrors:** an **eruption barrage** in the miniboss block (leash
> section of stepEnemies): every 5–7.5s within 750u, two telegraphed bursts —
> one at the ship's position, one leading its velocity (+0.95s) — 1.1/1.35s
> red rings (drawn unscaled in drawEnemy, they land at the HELM not the boss),
> then strike r80 for 18. Its kind's roar at 0.45 is the audio warning.
> **Whirlpool loop** louder + wider: 0.95 inside the funnel, ramp from 300u off
> the rim (was 0.55 max/220u — inaudible under the mix, the user's report).
> Verified: shockwave 24dmg+knockback in range / 0 at 600u; eruption 18 dmg
> stationary / 0 when dodged; shot dmg 15/20; fresh sloop 70hp.
> **Test gotcha:** never scan ±30 chunks of uncached world in an eval
> (~3.7k genChunk calls wedges the tab for minutes and the CDP session with
> it) — find POIs via mapPois() or keep scans ≤ ±8 chunks.
>
> **SPRITES RETIRED + CODE-ART VARIETY PASS (2026-07-03, v0.17.4–v0.18.2 — 4
> verified batches, console clean, saves reset):** the user saw the wired land
> sprites in play and called it — too detailed against the code-drawn world,
> keying artifacts at game scale; **"go back to just the simple code assets and
> refine those + make more of them + make them better."** So:
> (1) v0.17.4 **retirement** — `LAND_SPRITES{}` emptied (pipeline + a few call
> sites stay dormant, ghost-ship pattern), `assets/sprites/` git-rm'd; all of it
> recoverable from v0.17.0–v0.17.3.
> (2) v0.18.0 **flora v2** — the drawIsland tree loop rewritten with hash-picked
> silhouette variants + flat 3-tone shading + terrain-style outlines: green =
> round oak / 3-lobe cloud / tall poplar (lit + shadow lobes); tropic = curved
> leaning palm trunk + 5–6-frond fan + coconuts; dead = bent trunk + wind-swept
> forks; ice = 2–3 stacked pine tiers w/ snow on each apex; desert = saguaro w/
> 1–2 elbow arms, sunlit ridge, rare pink bloom. New `lift(c,m)` lighten helper
> beside `quant()`. Render 2.0ms @ span 420 (fine).
> (3) v0.18.1 **port hamlets** — every port grows 1–2 smaller culture dwellings
> (`drawHut`, seeded by port seed, `landAt`-guarded off water) + dockside
> crates/barrels at the pier root (`drawCrateP`/`drawBarrelP`); building polish:
> euro sunlit roof ridge · norse roof plank seams · east lacquered corner posts ·
> desert sunlit dome edge. The dormant sprite block was removed from
> drawPortHouse (superseded).
> (4) v0.18.2 **sea stacks v2** — rock islets are now crooked faceted spires:
> 3–4 shrinking slabs on a seeded random walk, per-face directional shading,
> **dark plug pass underneath** (crooked slabs open back-face gaps that read as
> glass — diagnosed by sampling buffer pixels), **dedicated bare-stone palette**
> (biome cliff colours made ice-water stacks look like glass cones), snow cap in
> ice waters / gull-bleached cap on ~half elsewhere. Collision/shelf/foam still
> ride the island polygon.
> **RULE going forward: Helmsong's art is code-drawn.** Generated-sprite
> integrations have been retired twice now (green ghost ship, this land set) —
> refine and diversify the code art instead of importing images.
>
> **CUSTOM LAND-ASSET SPRITES (2026-07-03, v0.17.0–v0.17.3 — 4 verified batches,
> console clean, saves reset for a clean fresh start):** all 58 FLORA/Nano Banana
> sprites (user-generated per `docs/SPRITE_LIST.md`, sources in `~/Desktop/Land
> sprites Helmsong/spr-*/`) processed by a PIL script (scratchpad flow): corner-
> sampled magenta key → alpha, **soft alpha + defringe restricted to a 2px edge
> band** (v1 keyed a wide distance band and bleached magenta-ambient-lit
> interiors — the fort came out white), trim, LANCZOS downscale, saved to
> `assets/sprites/<kebab>-<n>.png` (mirror to /tmp on deploy, as ever).
> **Wiring (one draw fn per batch):**
> (1) v0.17.0 **trees** — `drawIsland` flora loop tries
> `drawSpriteBuf(sprVar(TREE_SPR[biome], hash2(tr.x|0,tr.y|0,53)), …, th*1.9)`;
> green/tropic/dead/snow/cactus, all 5 biomes + night tint verified. **Buffer
> vs crisp decided here:** the crisp overlay reads as smooth stickers on the
> chunky world (and always draws over ships) — buffer wins.
> (2) v0.17.1 **culture architecture** — `drawPortHouse` draws the house sprite
> (euro ×4 / norse ×4 / east ×2 / desert ×3, variant by port hash) + side-props
> (norse rune stone ×2, desert obelisk) along the shore. **Cut the open-frame
> pagoda take** (was house-east-2) — too wispy at game scale.
> (3) v0.17.2 **forts** — sprite keep (s*2.6 tall) + razed ruin (s*2.1); faction
> banner + siege hp bar + rubble smoke stay code-drawn, banner/bar anchors moved
> to the sprite tower tip (`bt` 1.5→2.4). **Two asset lessons:** the razed ruin
> shipped near-white w/ harsh black marks (read as static — toned down in PIL),
> and **canvas bilinear downscale aliases past ~2×** — fort assets pre-shrunk
> 256→128px so draw-time scaling stays clean. Verified intact/damaged/razed.
> (4) v0.17.3 **rocks & props** — sea-stack islets (`isl.rock`) draw the spire
> sprite from the foam (skip the polygon body), peak crags, icebergs (over the
> code foam halo, variant by berg seed), flotsam, loot crates/barrels (variant
> keyed on `L.ph` — x/y drift each frame and would flicker). Coin pouches stay
> code-drawn. Also brightened the dark house-euro-4 timber cottage ×1.3 (read
> as a dead tree at default zoom).
> **What stays code-drawn by design:** islands/terrain, piers, all flags/banners
> /pennants (animated + faction-recolored), ships/monsters (rotate → different
> pipeline), coin loot. Fallback code art remains at every site — delete a PNG
> and the old art returns.
>
> **VERSION CONTROL + SAVE SLOTS (v0.15.0):** the project is now a **git repo**
> (baseline commit = the playtest build; commit each verified batch with a
> summary — `git log` is the changelog now). **`GAME_VERSION = '0.15.0'`**
> (top of the save section — bump on notable batches; shown on the intro footer
> and stamped into every save with `savedAt: Date.now()`). **4 save slots**
> (`helmsong_save_1..4`, active pointer `helmsong_slot`, `activeSlot` +
> `setActiveSlot`): saveGame/loadGame/__HS.reset/pause-erase all key off the
> ACTIVE slot; binds + opts stay global (shared across captains). The intro's
> Continue/New buttons are replaced by a **slot picker** (`buildSlots()` — DOM
> cards: boat · day · coin · region · version ('older build' tag on mismatch) ·
> saved date · ✕ erase with confirm; empty = 'begin a new voyage'); Enter/Space
> resumes the MOST RECENT save by savedAt. Legacy `helmsong_v1` saves migrate
> into slot 1 once (tagged 'pre-0.15') and the old key is removed.
> `__HS.slot(n?)` reads/sets the active slot. NOTE: saves from before the
> continents batch predate the current world layout — fine to load but the
> charted world won't match; recommend fresh voyages.
>
> **MARKET SCROLLING (user: 'rows feel really tight — scroll past 15'):** the
> goods list now uses a FIXED 26px pitch and windows instead of squeezing —
> `ui.marketScroll` offset, `visRows` computed from panel height (~13 at 720p),
> `list.slice(scroll, scroll+visRows)`; dim centred hints top/bottom ('▲ N more
> above — scroll' / '— top/end of the ledger —', 34px reserved when scrollable).
> **Mouse wheel routes to the ledger while docked** (wheel handler gained a
> `ui.port` branch — this ALSO fixes the old bug where scrolling in port zoomed
> the world behind the panel); `ui.marketScroll` resets in openPort and clamps
> in draw. Buttons are built per visible row each frame, so Buy/Sell on scrolled
> rows bind the right good (verified: bottom-row buy → amber). The **port toast moved**
> from (px+64, py+ph−30) left-aligned — it was drawing behind the new price
> buttons — to right-aligned at (px+pw−64, py+ph−66), clear of both the strip
> and Set sail (very long toasts can brush the last tile, transient). The
> I-screen Hold rack still sells crates too — two doors, same shop.
>
> **NPC SELF-DEFENSE + LAND AVOIDANCE (user reports; verified, console clean):**
> (1) *'NPCs don't defend themselves'* — `hitNpc(n,dmg,src,by)` gained an
> attacker param: any hit records `n.foe = by` for 14s (`n.foeT`); plumbed at
> the 4 entity-known sites (rival shots pass `s.src`, raider ram + serpent
> strike pass `e`, jelly sting passes `e`). In stepNpcs the old navy-only prey
> block is generalized: **navy** fights by scan (unchanged), **merchants/
> traders** fight their foe with slower guns (reload 3.6/3.0 vs navy 2.4, led
> shots, `enemyFireA(...,false)` so their iron hits rival-faction hulls),
> **fishers** crowd on sail and FLEE their foe at 1.6× speed. Player attacks now
> also set `hostile=true` on merchants/traders (not just navy) — armed crews
> answer you too.
> (2) *'Ships get stuck on islands / drawn under them'* — new **`steerClear(o,
> dx, dy, look=130)`**: probes the intended course at look/2 and look; if land,
> tries bearings ±0.6/±1.2/±1.9 rad and returns the first clear one (or backs
> out). Wired at EVERY AI movement site: enemy ships, deepmaw, kraken (look
> 90), serpent hunt (100), npc hostile/defense/cruise blocks (cruise also
> re-rolls its target if truly pocketed). **`keepOffLand` rework:** eject
> margin 14→20 and velocity converts to a tangential COAST SLIDE (60% of speed,
> signed by approach direction) instead of the old ×0.3 damp — grinders now
> slip around coasts instead of sticking. The wider margin + no coast-hugging
> also mostly cures ships being overdrawn by island art (the y-sort draws
> islands at centre-y over ships pressed against their upper coast — proper
> per-polygon depth is still an open wrinkle, see §6 island-sprite note).
> **`audition.html`** (Desktop + /tmp copies, regenerated from the sfx dir by a
> bash one-liner) = click-to-play buttons for every file. v2 has FEEDBACK:
> green ring while playing, red ring + status line on load failure (v1 failed
> silently — the user clicked while no server was up and heard nothing).
> **GOTCHA: the preview server only lives while the assistant session holds it**
> — it dies between turns. For the USER to listen reliably: run their own
> `python3 -m http.server 9000 --directory ~/Desktop/Helmsong` (serves the live
> Desktop copy — game AND audition page, always current), or double-click the
> Desktop audition.html (file:// HTMLAudio works). Regenerate the page when new
> files land.
> **Level tune (user: 'too loud now'):** global **`SFX_GAIN = 0.42`** multiplies
> every file play (both the Web Audio path and the HTMLAudio fallback) — the one
> knob for the whole Suno set, which is mastered hot (~0dB peaks). Cannon output
> peak measured 0.61 → 0.17. Per-call vols in the SFX registry stay as RELATIVE
> balance; use SFX_GAIN for overall level.
>
> **⚓ THE BIG PLAN: see [`EXPANSION_PLAN.md`](EXPANSION_PLAN.md)** — the user's full
> second-era wishlist (factions, physical storms, organic islands, forts, fleet/empire,
> culture-themed regions, ammo types, loot drops, inventory, region goods, hazards…)
> grouped into dependency-ordered phases **V9 → V14**, with a traceability table and
> open questions. Agreed order: V9 Living Sea → V10 Gunnery & Spoils → V11 Trade Winds
> → V12 Banners of the Sea (factions) → V13 Many Flags (cultures) → V14 Your Flag
> (player empire). NOTE: the user's island reference images never attached — ask for
> them before starting V9's island rework. The items below predate that plan.

**✅ DONE — the visual-variety pass (3 batches, all verified error-free):**
1. **Distinct ship sprites per type & class** — replaced the single `RIM`/`WL` hull with a data-driven `VESSELS{}` registry (per-kind hull polygon + `wl` + deck height + masts + rig + bowsprit + feature flags) rendered by shared helpers `drawVesselBody()` / `drawVesselRig()` / `sqSail()`. Both `drawShip()` (player) and `drawEnemyShip()` (AI) call them; kind = `player.shipClass` or `e.type`. Silhouettes: sloop (fore rig), cutter (narrow + long bowsprit), merchant (fat galleon + sterncastle + 2 square sails), frigate (long + gunports), fisher (smack + net boom), warship (3 masts + gunports), raider (ram + low fore sail), ghost (tattered square sails), patrol (navy frigate hull). **Square sails belly forward** so they read from the iso top-down at any heading.
2. **Kraken** (`kraken:true` enemy) — a radial multi-arm grappler, distinct from serpent/boss. 260 HP, `drawKraken()` = purple mantle + glowing eyes + 8 writhing tentacles (stroked underlay + beaded taper). AI state machine `lurk → reach → grab`: one arm reaches for the ship; if it connects (<185, reach>0.7) it **grabs** — hauls the ship in, slows it, rakes ~10 dmg/s for 2.2s, then releases on cooldown. Loot: coin + scales + pearls. Spawns in monster regions (Cursed Deep) + rare in high-danger water via `pickEnemyType`.
3. **Wildlife sprite polish** — rewrote `drawWild()` whale/shark/dolphin/turtle/sprite with a heading-relative `wp(fwd,side,z)` helper: whale (tapered body + fluke + back sheen + eye + spout), shark (pointed body + dorsal + swept caudal fin + snout wake), dolphin (leap arc + dorsal + fluke + splash ring), turtle (shell + 4 paddling flippers + head + scute lines), sprite (glow + trailing motes). Birds unchanged.

> **NOTE:** the game has grown far past the original "visual-variety pass" — see the dated **Post-V7 polish** bullets in §3 for the full running log (world rework, HUD/pause/map/waypoint, bounty/economy, custom-sprite pipelines, new ships, etc.). The items below are the *current* forward list.

**Custom art — the decided approach (important):**
- The camera is 3/4-iso with real height. A **single flat top-down sprite rotated in 2D looks broken** — do NOT use it for ships. Two working pipelines exist:
  - **Directional sprites** (`drawDirSprite`, `SPRITES_DIR`) — 8 iso frames per object, pick-by-heading, un-rotated → keeps 3D look. Proven with the **green ghost ship** (since retired by user preference — ghost is code-drawn again; pipeline + PNGs remain, see §3). This is the way to do custom ships/monsters.
  - **Single flat sprite** (`drawSprite`, `SPRITES`) — ONLY for non-rotating iso assets (islands, ports, props).
- **User's current lean: keep ships code-drawn and add variety in code** (galleon/brig/trader/manowar, then corsair/dread). Custom directional sprites are available if they want them later.

**Open / next ideas:**
- ✅ **DONE (this batch):** code-drawn **corsair** + **dreadnought** ships; **2 more sea monsters** (angler lurer + jelly bloom); **2 more water-hazard regions** (Sargasso kelp + Verdigris toxic) with gate components. See the "Content batch" bullet in §3.
- Still open: a **size/detail bump for the frigate & merchant player classes** (they're the earned end-game hulls but only modestly bigger than the sloop). Also a `trader`/`brig` were already added earlier.
- **Island/prop sprites:** when the user provides 3/4-iso island sheets, wire them via the `drawSprite` pipeline — the one wrinkle is depth-sorting crisp island sprites correctly against the code-drawn ships (a boat south of an island must draw in front).
- **Even more sea monsters** if wanted (now have serpent/deepmaw/kraken/leviathan/angler/jelly) — e.g. a giant crab/reefback, a rogue-wave elemental, a whirlpool/maelstrom.
- Expand **water-hazard types** further (OSRS ref): crystal-flecked, rapids/whirlpool currents — the `waterGate()` + `REGIONS[].gate` + component scaffold is proven (2 new regions this batch); add more region types + components the same way. Note: gate placement adds one `haz` fbm field in `regionAt` — reuse/extend it rather than adding a new field per region.
- Asset-processing note: Nano-Banana exports arrive as **RGB with a baked checkerboard** (no true alpha). Throwaway PIL scripts flood-fill/global-key the checker → alpha (+ optional hue recolor, grid-slice). Ask the user for true-alpha exports when possible.

**House rules the user cares about:**
- **No multiplayer** (explicitly declined).
- **Art is code-drawn.** Generated-sprite integrations were retired twice (ghost ship, the v0.17 land set — "too detailed, look just wrong"). Add variety by refining the code art, not by importing images.
- Keep the lofi/chill tone: sailing pleasant by default, danger when you *choose* to linger/fight.
- Verify every change in the preview and keep the console error-free; reset the save so a fresh start is clean.

---

## 7. Conventions

- Match the surrounding code style (dense, single-line helpers, `col()/qcol()/tintC()` for palette, `polyFill()` for shapes, `proj()` for world→buffer).
- Palette colours are base arrays; environment tint (day/night/weather) is applied per-draw via `tintC`. Region ocean colour is spatial (see `drawWater`).
- Big features were shipped as batches (V1…V7); each verified before moving on. Continue that cadence.
