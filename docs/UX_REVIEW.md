# Helmsong — UX/UI design review (all screens & interactions)

> Session 2026-07-02, post-QA build, reviewed at 1280×720 (the common case) with
> spot-checks at 900×600. Lens: affordance, feedback, hierarchy, consistency,
> readability. Every screen walked with screenshots; interactive claims verified
> with real clicks. Console clean throughout.

## What's working (keep)

The ornate chrome is cohesive and legible; port tab hierarchy is clean (gold title →
sans lore line → tabs → content); the Quests tab's step states (✓ green / ▸ cream /
dim future) read instantly; context prompts carry the whole verb language of the game;
the chart's header teaches its own controls; danger banners (shoal/gate/squall) are
well-placed and appropriately alarming; night + lantern is both beautiful and readable.

## Findings → fixes (severity order)

> **STATUS: ALL 9 FIXED & VERIFIED (same session, console clean).** UX-1 — slots are
> real controls now (`ACTION_SLOTS[].click` + `actionBarRect`/`actionSlotRects` hit-tested
> in mousedown BEFORE fireCannon; hover ring on clickable slots); verified lantern/chart/
> mute clicks act, cannon+anchor slots and bar dead-space swallow, open-water clicks
> still fire. UX-2 — design height 700; US=1.0 at 720p (crisp), short windows still
> scale. UX-3 — legend rebuilt: 7 region swatches + 'ports & forts fly their owner's
> colours'. UX-4 — Logbook 'YOUR FLAG' section (banner chip · stronghold · fleet +
> standing chips) with achievements two-columned to pay for the space. UX-5 — upgrade
> pitch 30→29 + crest chips fnt(10)/3-char middle-baseline; shipyard clears the footer
> at 696px. UX-6 — labelled backed hull bar beside Dismiss + 'hire another escort:'
> microcopy. UX-7 — 'Pay 250c' / 'Sign · 400c'. UX-8 — fortress is 'stronghold'
> everywhere (chart label, guide arrow); 'Hold' = cargo only. UX-9 — journeys show
> 'begins: <first step>'. BONUS: catch-toasts were peeking out from behind the open
> chart at both edges — now hidden while the map is open.

### UX-1 — CRITICAL (affordance): action-bar slots look clickable — clicking fires the cannon
The seven hotbar slots have button framing, recessed slot art, and hotkey chips —
every visual signal of "tap me." Verified: clicking the lantern slot fires a cannon
shot at the water (mousedown falls through to `fireCannon`). Worst case, a hotbar
misclick hits a nearby merchant → accidental piracy, bounty, faction damage. The
V-era note said "display-only, to avoid stealing the fire input" — but the current
result is the worst of both worlds.
**Fix:** hit-test the action-bar rect in the mousedown handler *before* `fireCannon`:
clicks on slots trigger their action (net→`handleAction`, lantern→`toggleLantern`,
chart→`toggleMap`, hold→`toggleInv`, mute→`toggleMute`; cannon slot and anchor
swallow), any other click within the bar panel is swallowed. Add a gold hover ring on
clickable slots so the affordance is honest.

### UX-2 — MAJOR (regression from SD-4): the whole UI is blurry at 720p
`US = min(1, H/740)` puts a 1280×720 window at 0.973× — every VT323 glyph is
fractionally resampled and visibly soft, betraying the pixel aesthetic at the game's
most common size. Verified against pre-SD-4 screenshots.
**Fix:** design height 740 → **700** (`US = clamp(uiScaleMul * Math.min(1, H/700),
0.7, 4)`). 720p returns to native/crisp; genuinely short windows still scale. Requires
the UX-5 shipyard trim below (the 700px budget is already pixel-tight).

### UX-3 — MAJOR (stale information): the chart legend is wrong twice
Legend reads "■Calm ■Coral ■Desert ■Cursed ■Port": (a) it's missing four of the
eight tinted regions (Ice, Sargasso, Verdigris — added V8+; Open Sea untinted);
(b) the 'Port' swatch is the pre-V12 orange, but port dots now wear faction colours —
which are explained nowhere. Fort squares, loot glints, and contested rings are also
unexplained, but the port colours are the active misinformation.
**Fix:** rebuild the legend line with all seven region swatches, and replace the Port
entry with the note "ports & forts fly their owner's colours". (Full symbol glossary
can wait; correctness can't.)

### UX-4 — MEDIUM: the Logbook predates your identity
The "who am I" tab still shows only V7-era stats/ship/achievements/lore. Your banner,
stronghold, fleet, and faction standings — the entire V12–V14 identity — are absent
(standings appear ONLY inside the Harbour tab, where nobody looks for a status
summary). Half the tab is empty space.
**Fix:** add a "YOUR FLAG" section to the Logbook: banner name + colour chip (or "no
banner raised"), stronghold status, fleet roster one-liner, and reuse the Harbour
standing-chip row.

### UX-5 — MEDIUM: shipyard bottom row collides at 696px + illegible crest chips
At 720p (design 696px after UX-2's fix) the Crest cosmetics row and the footer toast
overlap, and the crest chip labels truncate to 4-character garbage ("A0dh") at that
size. The tab was budgeted to exactly 700px.
**Fix:** trim upgrade row pitch 30 → 29 (frees 10px) and render crest chips at
3-char + fnt(10) or swap the text for a tiny glyph. Verify at 696 and 700.

### UX-6 — MINOR: fleet rows in the Crew tab
The escort hp bar floats mid-panel with no backing, label, or association — it reads
as a stray green line. The escort purchase row has no header, so four price buttons
appear unexplained.
**Fix:** dark backing + right-align the bar near the Dismiss button; add microcopy
"hire another escort:" above the buy row.

### UX-7 — MINOR: diplomacy buttons are bare prices
"250c" / "400c" as button labels don't communicate the verb; everywhere else buttons
lead with actions (Buy, Sell, Hire, Take).
**Fix:** "Pay 250c" and "Sign · 400c".

### UX-8 — MINOR: "hold" means two different things
The cargo tab is "Hold"; your fortress is "your hold" on the chart label and guide
arrow but "your stronghold" in the prompt and panel title. One word, two systems.
**Fix:** standardize the fortress as "stronghold" everywhere player-facing (guide
arrow "⌂ stronghold", chart label "your stronghold"); "Hold" remains cargo-only.

### UX-9 — NIT: journeys are accepted blind
Captains' Journeys rows show title + captain + reward but nothing about what the
journey demands until after you Take it.
**Fix:** render the journey's first-step text in dim sans under each unaccepted row
(space exists; the tab is half empty).

## Suggested batch order
UX-1 first (it can cause piracy from a misclick), then UX-2+UX-5 together (same
layout budget), UX-3 (misinformation), then UX-4/6/7/8/9 as a single polish sweep.
