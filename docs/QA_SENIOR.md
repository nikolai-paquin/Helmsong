# Helmsong — Senior-dev QA pass (edge cases, regressions, soak)

> Session 2026-07-02, post new-player-fix build. Scope: the two watchlist items nobody
> had re-tested (Deep Song chain, Tidebound stronghold), adversarial edge cases (world
> wrap, poles, UI state collisions, awkward-moment saves), an 8-sim-minute soak for
> leaks, and small-window layout. Console clean through every test.

## Findings (severity order)

> **STATUS: ALL FIXED & VERIFIED (same session, console clean).** SD-1 — quest played
> to step 3 the real way; the Leviathan rose unprompted ~9s after entering the Deep.
> SD-2 — `wrapDX`/`holdDist`/`holdNearestX` helpers; verified one world east AND two
> west: waters 'you', dist 336 (not 80k), Space enters the hold while wrapped, guide
> targets the near period copy. SD-5 — autosave skips while sinking + loadGame clamps
> hp to ≥20% max (synthetic hp:0 save loaded at 20). SD-4 — `computeUIScale` now
> `min(1, H/740)` (floor 0.7); verified at 900×600 the 18-row worst-case market fits
> with margin, and at 1280×720 (US 0.973) chart clicks, inventory, and waypoints all
> land. BONUS hardening found during verification: the emulated viewport can change
> without firing a window resize event, leaving a stale canvas — `frame()` now
> self-heals (`if (W!==innerWidth||H!==innerHeight) resize()`), which also covers
> mobile orientation quirks.

### SD-1 — CRITICAL: the quest Leviathan can never spawn
The guaranteed boss spawn (index.html:4207) requires `player.quest.step === 2` while
inside the Cursed Deep — but *entering* the Cursed Deep advances the quest 2→3 on the
same frame (`stepDanger` fires the 'cursed' event immediately). By the time the 8s
boss timer expires, the condition is permanently false; the only remaining path is
the post-quest 0.09% random roll. **The Deep Song soft-locks at "Slay the Leviathan"
for every normal player.** (Likely regressed when V-era playtest fixes made the
Cursed Deep entry event immediate; the kill handler at :4129 correctly expects
step 3.) The rest of the chain is healthy — with a debug-spawned boss, the 3-phase
fight, Heart award, delivery, and +800 reward all completed.
**Fix:** change the spawn condition to `player.quest.step === 3`. One token.

### SD-2 — MAJOR: your stronghold doesn't survive circumnavigation
Sail one full world east (the world wraps; coordinates stay continuous) and arrive at
the *identical* island: `waters()` reports Crown, the "enter your stronghold" prompt
never appears (distance reads 80,180), and the guide arrow points a full world away.
Fort *state* is period-stable (keys derive from the wrapped column — verified), but
every `player.hold` proximity check uses raw `Math.hypot`.
**Fix:** add a wrap-aware distance helper (the pattern already exists in `ledgerDist`)
and use it in `homeWaters`, `handleAction`, the context prompt, and `drawGuides`;
the guide arrow should target the nearest period copy
(`hx = hold.x + round((ship.x-hold.x)/WORLD_W)*WORLD_W`), like waypoints already do.

### SD-4 — MINOR (layout): short windows overflow the market panel
At 900×600 the market list (19 rows worst case: 16 staples + local exotic + owned
exotics) runs past the panel: the last row's buttons sit under Set-sail and the
footer toast overdraws its text. The adaptive pitch already floors at 20px — the
panel's 700px design height simply doesn't fit in H−24.
**Fix:** use the dormant whole-UI scale shipped in the font-scale era —
`computeUIScale(): US = clamp(uiScaleMul * Math.min(1, H/740), 0.7, 4)` — so ALL
chrome scales down on short windows. `hit()` and `mapClick()` already divide by US;
the infrastructure was built and verified, just pinned to 1.

### SD-5 — MINOR (edge): autosave during the sinking animation
The 4s autosave can fire mid-sink and persist `hp: 0` (the `sinking` flag isn't
saved). Reloading that save yields a 0-hp, not-sinking ship: unkillable limbo until
the first scratch, and the death penalties never applied.
**Fix (both cheap):** (a) skip autosave while `ship.sinking` in `frame()`;
(b) belt-and-braces in `loadGame`: `ship.hp = clamp(s.hp ?? maxhp, maxhp*0.2, maxhp)`.

### SD-3 — WITHDRAWN (test artifact, worth recording)
"Ice gate doesn't drain" — my own harness had left the port panel open from a
previous step; environmental drains correctly pause while docked (`!ui.port`).
Re-verified clean: −5hp/s drain vs +2 repair = net −3/s in Frostreach without the
Icebreaker. Lesson repeated from the V12 notes: multi-step preview tests must manage
UI state explicitly — `dockAt()` leaves the port open.

## Verified clean (no action)

- **Deep Song steps 0→4** (dock, rare catch, cursed entry, boss fight via debug
  spawn, Heart, delivery, reward) — everything but the SD-1 spawner.
- **Tidebound stronghold**: always-hostile, ghost garrison sortie, relics+pearls
  spill, −tide rep on razing.
- **Pole wall**: hard clamp at the ice, no NaN, no unfair damage.
- **Seam crossing**: sailed x → past WORLD_W under power; regions continuous, no
  land pop-in, no NaN. (Entities all use continuous coordinates by design.)
- **8-sim-minute soak** under war + storms + escort + combat: every transient pool
  bounded or shrinking (loot 21→2, fx 44→7, storms capped 2, caps respected);
  persistent maps grow only with world coverage; 8 sim-min in 1.66s wall (~60× real
  time headroom); no NaN.
- **UI state machine abuse**: 14-step rapid M/I/Esc storm resolves to a clean state;
  keys typed inside the naming dialog stay text (no screen toggles); Esc unwinds
  overlays in the right order.
- **Mid-state saves**: fleet/hold/banner/ledger/stock all round-trip; only the
  mid-sink hp persists wrongly (SD-5).

## Suggested fix order
SD-1 first (one token, unblocks the whole main quest), SD-2 second (four call
sites + one helper), then SD-5 (two one-liners) and SD-4 (one function, but eyeball
every screen at 900×600 and 1280×720 after enabling the scale).
