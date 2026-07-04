# Helmsong v0.18.9 — Four-Lens QA Report (2026-07-03)

> Method: one hands-on playtest driven with real inputs in the preview (fresh
> newbie session → fishing → docking → trading → pirate duel → terror fight
> with a maxed frigate → save/load parity), plus two parallel deep audits:
> a systems/balance code audit (master-game-dev lens) and a UI/UX + newbie
> heuristic audit. High-impact claims were re-verified against the code before
> inclusion. Console stayed clean through the whole hands-on session.

## The three findings that matter most

### P0-1 · Terror/boss balance is broken at BOTH ends
- **Endgame melts them (hands-on, verified):** a maxed frigate (66 dmg × 2
  cannons, ~0.7s reload ≈ 200+ DPS) killed the 540hp Reefjaw in **3.0 seconds
  without taking damage**. The 600hp Leviathan dies in ~3.3s of the same fire.
  The stock→maxed DPS spread is ~8×, so flat hp can't serve both ends.
- **They can't even reach a careful player (systems audit, verified):** enemy
  shots die at ~296u of travel (vz 66, gravity 150), but the terror deepmaw
  spews from up to 380u — park at 300–380u and its ranged attack physically
  cannot land. Same class of bug: **dread keepDist 350 / galleon 320 vs shot
  reach 296** — the two biggest pirate ships deliberately hold station beyond
  their own range and splash every volley short.
- **And they ambush newbies with ~2s of warning (UX audit):** terrors are
  assigned to the friendliest-reading seas (tropic "Coral Reaches", desert),
  arm ~30–60s after region entry with no progression gate, and announce via a
  single 1.9s toast that can queue behind fish-catch chatter. A 70hp sloop
  dies to 3 eruption hits; the tuning verify was literally "parked stock sloop
  dead in ~5s."
- **Two cheeses:** escorts are invisible to all monster attacks (grab/strike/
  eruption test only `ship`/`npcs`), so 2 escorts = a zero-input terror kill in
  ~47s; and a rival faction landing the killing blow skips the rest timer AND
  the purse (`killEnemy` src early-return) — instant respawn, no loot.

**Recommended fix (one batch):**
1. Fix enemy ballistics so shot reach ≈ fireRange (compute vz from target
   distance like fort shots lead their angle), or clamp keepDist/fireRange ≤270.
2. Raise terror hp to ~1,400–1,800 and Leviathan to ~2,000 (they are endgame
   content; a stock sloop is not supposed to win anyway), AND/OR give terrors
   a dive-reposition invuln (~3s) at 66%/33% hp to break burst DPS.
3. Newbie telegraph: persistent flashing banner ("⚠ the water trembles —
   something vast rises. Make sail!") 8–10s before the terror spawns, spawn it
   further out, delay its first volley ~5s.
4. Let eruptions/melee damage escorts (damageEscort exists).
5. Hoist bossCd/bossRest above the `src !== 'player'` early-return in killEnemy.

### P0-2 · "Danger communication" needs to be a system, not toasts
The game teaches its *systems* beautifully (R-swap discoverable 3 ways,
slower-is-better fishing prompt, quest tracker, abandon button — all verified
still intact). The teaching debt is concentrated in *danger*:
- **Gate water gives no warning before it bites** — no region-entry toast; the
  banner appears only once you're already draining 5hp/s at half speed. The
  chart legend names the seas but never says they damage you.
- **The toast queue has no priority** — 4-deep FIFO, 1.9s each; "the Reefjaw
  rises!" or "Piracy!" can sit behind 3 fish toasts (~7.6s) or be dropped.
- **Unattributed drains** — fire-aboard (4hp/s) and whirlpool-core (6hp/s)
  have no banner text; in overlap the player reads "the game is cheating."
- **WANTED banner never says how to clear it** (payoff guidance is a single
  missable toast at crime time).
- **Esc on the first-run naming dialog silently accepts** the random name with
  no hint that renaming exists (Shipyard).

**Recommended fix (one batch):** an `urgent` flag on setToast (replaces current
toast, 3s, red tint, never dropped) + one-time gated-region entry toasts +
skull glyph on gated legend swatches + banner lines for burn/whirl + a second
line on the WANTED banner ("pay off at any Harbour · sinking pirates helps") +
naming microcopy ("Enter or Esc to christen her · rename later at any Shipyard").

### P0-3 · Delivery contracts break after circumnavigating (wrap bug)
`completeDeliveries` checks raw `Math.hypot(dockX - c.tx, …) < 50`, and the
guide arrow + chart pin also use raw `c.tx`. Coordinates stay continuous when
you sail around the world, so after one E–W loop (~8–10 min) the delivery at
the *correct* port silently fails and the arrow points a full world-width
backwards. Waypoints/stronghold/ledger already snap to the nearest period —
contracts were missed. **Fix:** apply the existing `wrapDX` pattern in
completeDeliveries and snap `c.tx` to the ship's nearest period for guides/charts.

## P1 — significant, fix soon

1. **Respawn death loop in gated water** — sink while your lastPort is inside
   a gated sea without the component → respawn into 5hp/s drain at half speed
   on a fresh 70hp hull; each loop costs another 10% coin + 25% cargo. Fix:
   waterGate respects graceT, or respawn falls back to the nearest non-gated port.
2. **One stray cannonball on a neutral = instant WANTED** (+50 bounty on first
   *hit*, not kill). In a lane fight a missed shot clips a crossing merchant
   mid-combat. Fix: first unprovoked hit = warning toast + NPC flees; second
   hit converts. Kills stay instant.
3. **Money loops (ranked by degeneracy):**
   - *Chum fishing*: chum adds flat richness with no stock drain and works in
     the port no-spawn bubble → risk-free, order-of-magnitude best income.
     Fix: chum multiplies ground richness (small bonus off-ground) + chum lure
     pierces the port spawn suppression.
   - *Red Sail fort farming*: fort guns reach ~249u, player reach 382u → a
     total-safety sniping band; razing pays 500–700c + rep UP with crown/guild
     and rebuilds in 3 days. Fix: fort shot reach ≥400u + diminishing plunder
     per rebuild (or Red Sail retribution).
   - *Merchant-hold-at-cutter-speed*: free class switching doesn't clamp cargo
     (load 80 on Merchantman, switch to Cutter, sail fast, sell). All addCargo
     callers survive it (verified) — pure exploit. Fix: block switch while
     `cargoCount() > new cap` ("offload first").
   - *Hail spam*: named captains pay 20c at 40% per hail, no cooldown. Fix:
     `n.hailed` flag per docking/day.
   - *Piracy pays too well*: rich trader ≈ 300–450c for +40 bounty payable
     1:1; rep floor only costs ~16% prices. Fix: bounty scales with plunder
     value; navy pressure escalates with lifetime piracy.
4. **UI traps:**
   - M or I inside the stronghold opens the chart/captain's screens *under*
     the hold panel — Esc then "does nothing" and Store/Take stop working
     (hidden inv swallows clicks). Fix: `|| ui.hold` guards on both toggles.
   - Rebinding allows duplicate keys (Lantern on W…) and collides with
     hardcoded O. Fix: swap-on-conflict + reject `o`.
   - Market `[Buy][Sell][Sell all]` — Sell-all vanishes at own==1 and the
     row shifts, putting Buy under a spam-clicking cursor. Fix: always
     reserve the Sell-all slot (dim it when not applicable).
   - Clicking the gold-ringed minimap labeled "CHART" fires the cannon at
     whatever is bottom-right of you. Fix: minimap click = toggleMap; swallow
     clicks on compass/pills/weather panel.

## P2 — perf, persistence, polish

- **Perf:** max-zoom local chart with maxed Spyglass scans ~12.7k chunks/frame
  (cap the POI radius or cache per chunk-crossing); drawWater re-samples ~2.4k
  regionAt cells + allocates ImageData every frame (reuse + scroll the grid);
  chunkCache is unbounded and keyed by continuous coords — leaks a full period
  strip per circumnavigation (LRU-evict or key by wrapCol).
- **Persistence:** storms/bergs/war/navy-hunters/weather.seq reset on reload —
  refresh = get-out-of-trouble button (persist weather seq + war + hunter
  count); save-time trims (visited 200 / flotsamTaken 250 / explored 400) can
  regress completionist state — raise or drop the caps.
- **Jellies don't count toward monster bounty contracts** (every other monster
  calls addBounty('serpent')) — one-line fix.
- **Colorblind:** faction identity is hue-only everywhere (1-letter initials
  next to port dots when the Factions overlay is on).
- **Small stuff:** ammo count on the cannon slot needs a dark chip behind it;
  ammo crate buttons say "30c" not "+5 · 30c" (verb-first rule); stronghold
  exit says "Set sail" while you're standing in your fort.

## What's genuinely good (verified hands-on)

- **Early economy pacing:** one fishing run (~4 min) funds the first upgrade;
  contracts net ~450c profit per 15-min round trip; ammo crates affordable in
  the first session.
- **Combat danger:** a fresh-sloop pirate duel cost 35/70 hull to win — spicy,
  fair, and loot + zero bounty flowed correctly.
- **Save/load parity is perfect** — position, ammo counts + selection, bounty,
  contracts, upgrades, boat name all restored exactly.
- **All 17 prior QA fixes (UX-1..9, NP-1..8) verified still intact — zero
  regressions.** No console errors at any point. addCargo negative-return
  class guarded at every call site. No NaN risks found in hot paths.

## Suggested fix batches (in order)

1. **"Terrors done right"** — P0-1 items 1–5 (ballistics, hp/dive, telegraph,
   escorts, rival-kill) + re-verify winnability at sloop/cutter/frigate tiers.
2. **"Danger speaks"** — P0-2 urgent-toast channel + entry warnings + banners.
3. **"Wrap + traps"** — P0-3 contracts, respawn loop, M/I-in-hold, rebind
   conflicts, sell-all shift, minimap click.
4. **"Honest economy"** — chum, fort farming, cargo switch, hail, piracy pay.
5. **"Longevity"** — perf trio + persistence + polish list.
