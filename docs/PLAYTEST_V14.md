# Helmsong — Full playtest report (V1–V14)

> Session 2026-07-02, build after V14 Batch 3. Fresh save, played through every major
> system in the preview (real key/mouse input where it mattered, `__HS` to compress
> travel). Console stayed clean the whole run — zero errors or warnings.

## What was played

1. **First minutes** — christening (suggested name, Enter), W/A sailing, full-sail cruise
   at 172 u/s, HUD reads. ✔
2. **Fishing → sell loop** — trawled a rich ground (12 catches / ~20s), sold at
   New Brackford for +142c. ✔ (but see B1)
3. **Exotic trade route** — bought 5 Amber at its Calm Shallows source (205c), sold at a
   Sunspine port for 104c each (+315 profit); ledger correctly recorded El Nubhet as the
   best amber market. ✔
4. **Gunnery** — chain/grape/fire/oil purchased in crates; brig fight; wreck spilled 3
   loot items, all magnet-collected (+66c). ✔ (see P1)
5. **Piracy loop** — merchant kill: bounty 90, guild rep −26, loot adrift, 2 navy
   hunters at notoriety 80, partial bounty payoff 90→50 with 40 coin. Waters toast
   flipped to "wary". ✔
6. **Death & respawn** — sank with an escort in tow; washed ashore fine, fleet survived.
   ✘ escort stranded (B2)
7. **Harbour tab, worst case** — bounty + tribute + 3 contracts + 3 news. ✘ two layout
   bugs (B3, B4)
8. **Squalls & gates** — full-sail inside a cell cost 29 hull in 5s, outran it cleanly;
   Cursed Deep gate drained as designed. ✔
9. **Performance** — war skirmish + storm + 5 enemies + 6 npcs + escort: 0.12 ms per
   sim tick. No concern. ✔
10. **Chart / waypoint / inventory / pause with real inputs** — M, click-waypoint
    (snapped correctly), I, Esc chains. ✔
11. **V14 systems** (verified during the phase, re-touched here): escorts fight and
    screen; stronghold claim/warehouse/guns; banner founding; tribute truce; guild pact
    + revocation. ✔

---

## Problems found → proposed fixes

> **STATUS: ALL FIXED & VERIFIED (same session).** Re-verified numbers: B1 — 14 catches
> now leave stock at 0.52 (was 0.17 at 12); B2 — escort 127u from the ship after respawn
> (was 2,500+ stranded; NOTE the first fix attempt called `syncEscorts()` BEFORE respawn
> moved the ship — it must run at the END of `respawn()`); B3/B4 — max-content Harbour
> screenshot clean, news clipped with '…' above the footer; P1 — chained brig reads at a
> glance (dark limp sails, no flag); P2 — '… and N more' shows past 13 rows; P3 —
> `st.fireT` held at 1.5 after first blood. Console clean throughout.

### Bugs

- **B1 — BALANCE: fishing drains grounds too fast.**
  12 catches in ~20 seconds took a rich ground from 0.9 → 0.17 stock (0.06/catch).
  One short session strips a ground bare, which reads as punishing in the chill loop —
  depletion should be felt across *sessions*, not within one net-cast.
  **Fix:** reduce per-catch drain `0.06 → 0.035` in `stepFishing` (a rich ground then
  supports ~2 full holds before thinning; NPC fisher drain unchanged; regen unchanged).

- **B2 — Escorts stranded on respawn.**
  After sinking, the player respawns at the last port but `respawn()` never calls
  `syncEscorts()` — the fleet is left at the sink site (measured 2,547u away) crawling
  back at the idle catch-up cap (~60 u/s ≈ minutes of waiting).
  **Fix:** call `syncEscorts()` inside `respawn()` (the survivors wash ashore with you).
  Also raise the idle catch-up cap `max(60, …) → max(110, …)` in `stepEscorts` so
  escorts rejoin faster after any long separation (e.g. after a fight drift).

- **B3 — Harbour tab: "all taken" overlaps MARKET NEWS.**
  In `drawHarbourTab`, the empty-offers branch (`if (!offers.length){ … 'all taken' }`,
  index.html:3574) never advances `ry`, so the MARKET NEWS header draws on top of the
  placeholder text. (The empty-contracts branch above it advances correctly.)
  **Fix:** add `ry += 26;` inside that branch.

- **B4 — Harbour tab can overflow its panel at max content.**
  Standing chips (62) + diplomacy (84) + bounty banner (62) + 3 contracts (84) +
  3 offers (96) + news (78+) ≈ 540px of content in a ~480px area — the news section
  can run into the Set-sail footer, worse on short windows (panel caps at H−24).
  **Fix:** (a) tighten pitches — contracts 28→24, offers 32→28, news 26→22; and
  (b) hard-stop: compute `footY` (as the market tab does) and stop drawing news rows
  once `ry > footY − 30`, appending a dim '…' if truncated.

### Polish (small, worth doing in the same pass)

- **P1 — Chain-shot has no visual feedback on the target.**
  A slowed enemy looks identical to a healthy one; players can't tell the chain
  connected. **Fix:** in `drawEnemyShip`, when `e.slowT > 0` darken the sail colour
  (`cols.sail × 0.7`) and skip the flag — reads as "rigging shredded" at a glance.

- **P2 — Stronghold warehouse hides rows past 13.**
  Both columns `slice(0,13)` with no indicator; with many distinct goods the rest are
  silently unlisted. **Fix:** draw '… and N more' in dim text under a truncated column.

- **P3 — First-aggro fort guns are instantly lethal at point blank.**
  Working as designed ("danger is sought out") — the guns + sortie sank the test sloop
  in ~15s at point-blank. Keep the lethality, but give one beat of readable warning:
  **Fix (optional):** on first aggro, the fort's first return shot gets +1.5s delay
  (`st.fireT = max(st.fireT, 1.5)` when aggro is freshly set) so the "You fire on a
  fort!" toast lands before the first ball does.

### Verified-fine (no action)

- Trade economy: no money loop found (pact + rep discounts cap well below arbitrage).
- Truce correctly excludes war-zone ships; war/grapple/slicks are transient by design.
- Escort shots can't commit piracy or hit forts; your fort's guns can't hit escorts.
- Second-ruin claiming correctly blocked (one hold by design).
- Save/load round-trips: fleet, hold, vault, banner, forts, stock, ledger, rep.

### Watchlist (not exercised this run)

- The Deep Song quest chain + Leviathan boss (untouched since V5, verified then).
- Tidebound stronghold sortie/loot variant (code-shared with tested crown path).
- Touch UI overlap with the action bar (pre-existing note, mobile-only).
