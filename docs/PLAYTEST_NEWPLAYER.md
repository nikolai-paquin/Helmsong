# Helmsong — New-player playtest (fresh eyes, no prior knowledge)

> Session 2026-07-02, post-fix build. Played a genuine first session: factory-reset
> save AND keybinds, used only what the screen teaches — real key/mouse input, no
> debug shortcuts except to measure. Console clean throughout.

## The session as it happened

Title screen → christened "the Dawntreader" (dialog self-explanatory) → the boat was
already under way, a ⚓ guide arrow named New Brackford, the prompt offered "Space to
cast net". Sailed to port on W/A/D alone; came in too fast and the **shoal warning
taught me to ease off** — a genuinely good organic tutorial beat. Docked with Space,
browsed all 7 tabs, left, cast a net from the prompt alone (3 fish in 26s). Wandered
into the Open Sea, got repeatedly jumped (see NP-2), somehow became **WANTED without
ever firing a shot** (see NP-1), watched a lovely readable night under the lantern,
picked a fight with a warship as a scared beginner and sank in 43 seconds — penalties
felt gentle, respawn clear.

**What already works for a newcomer:** the title-screen key list; the christening
dialog; guide arrows; every context prompt (dock/cast/fire/claim); the shoal-scrape
lesson; hotkey chips on the action bar; night readability; death being a setback,
not a punishment.

---

## Problems → proposed fixes (severity order)

> **STATUS: ALL 8 FIXED & VERIFIED (same session, console clean).** Re-verified:
> NP-1 — a Red-Sail fire burning a merchant down now yields bounty +0 / notoriety +0 /
> rep ±0 with the toast naming the true culprit ('lost to Red Sail guns'), while a
> player-set fire still credits the kill; NP-2 — the same 2.5-min Open Sea cruise drew
> 1 spawn (was 5), hull 99 (was 77); NP-3 — ✕ abandons a contract (−2 rep with the
> issuing faction, tagged at portContracts); NP-4 — '❖ Put in at any harbour' leads
> the top-right tracker (hudTopAvoid updated) + a logbook toast follows the christening;
> NP-5 — I and Esc rows on the title screen; NP-6/7 — prompt reads 'Space to stow the
> net · slower is better' (threshold 45, not 70 — net drag means trawling speed rarely
> passes 70, discovered in verification); NP-8 — 'Monsters slain'.

### NP-1 — BUG (high): you get blamed for fires you didn't start
Wandering the Open Sea without firing once, I ended the session **WANTED (40 coin)**
with guild rep dinged. Cause: a pirate's *fire shot* ignited a merchant during
NPC-vs-NPC combat, and the burn-tick death calls `killNpc(n)` with the default
`src='player'` (index.html:4746) — `applyShotFx` never records who set the fire.
The mirrored enemy path (`killEnemy(e)` on the enemy burn tick) silently *credits*
the player with kills, bounty-easing, stats, and rep they didn't earn.
**Fix:** pass the shooter through — `applyShotFx(t, kind, src)` stores
`t.burnSrc = src` on fire; both call sites in stepShots pass `'player'` (friendly)
or `s.fac` (faction shots); the two burn-tick deaths become
`killNpc(n, n.burnSrc || 'redsails')` and `killEnemy(e, e.burnSrc || 'player')`.

### NP-2 — BALANCE (high): the "trade lanes" aren't chill
Cruising the Open Sea (temperate, danger 1.0) passively for 2.5 minutes drew
**five hostile spawns** — two brigs, a corsair, a pirate, a serpent — and knocked a
non-fighting sloop to 77 hull. One ambush every ~30s in the region every trade route
crosses contradicts the pillar ("danger is sought out, not ambient"). The player can
outrun everything, but a first-hour player doesn't know that yet.
**Fix:** stretch the spawn clock in `stepEnemies` — `spawnT = (18 + rand*16) /
(danger*lure)` (was 11+rand*11) — and lower `REGIONS.temperate.danger 1.0 → 0.8`.
Leave cursed/toxic/high-danger water exactly as is; that's where danger belongs.

### NP-3 — TRAP (medium): contracts can never be abandoned
Delivery contracts pay the best early money (+475–777) but require buying the goods
up front (e.g. 8 Furs ≈ 472c) — impossible on a 60-coin start. A newcomer who takes
three such contracts has **all three slots dead forever**: contracts only leave the
list on completion (no abandon, no expiry).
**Fix:** an ✕ button on each YOUR CONTRACTS row (`player.contracts.splice`) with a
small flavor cost — say −2 rep with the issuing port's faction — so it's an out, not
an exploit.

### NP-4 — ONBOARDING (medium): the main quest is invisible until you stumble on it
Before the first dock, nothing anywhere mentions The Deep Song; after docking, the
step advance is a 2-second toast that's easy to miss behind the port panel, and the
top-right tracker lists *contracts only*.
**Fix:** (a) show the active quest step as the first line of the top-right tracker
('❖ Put in at any harbour', in the existing tracker style); (b) queue a one-time
toast on fresh start ("Your logbook marks a first task — put in at any harbour").

### NP-5 — STALE DOCS (low): title screen key list is missing V10+ keys
The intro lists W/S, A/D, Shift, L, Space, Click, M, P/O — but not **I** (Ship's
Hold, added V10) and not **Esc** (pause/menu, where Controls lives).
**Fix:** add two rows to the `#intro .keys` block: `I — ship's hold & ammo` and
`Esc — menu`.

### NP-6 — HIDDEN MECHANIC (low): the slow-trawl bonus is never communicated
Fishing is up to 3× faster when nearly stopped (`slowBonus`), but nothing says so;
a new player trawls at full sail and just thinks fishing is slow.
**Fix:** when fishing with speed > 70, extend the prompt: "Fishing… · Space to stow
the net · slower is better".

### NP-7 — WORDING (nit): "Space to haul in" doesn't haul anything in
It *cancels* fishing; catches land automatically. **Fix:** reword to "Space to stow
the net" (and see NP-6's combined prompt).

### NP-8 — LABEL (nit): Logbook counts every monster as a Leviathan
`stats.slainMonsters` increments for serpents, deepmaws, krakens, anglers, jellies —
displayed as "Leviathans slain". **Fix:** rename the row to "Monsters slain".

---

## Suggested fix order
NP-1 (attribution bug) and NP-2 (spawn cadence) are the two that actively damage a
first session; NP-3/NP-4 are the quality-of-first-hour pair; NP-5–NP-8 are one-line
text/UI touches that can ride along in the same batch.
