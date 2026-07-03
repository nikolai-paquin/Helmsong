# Helmsong — Land Asset Sprite List + Generation Prompts (FLORA / Nano Banana)

> **THE STYLE BLOCK — append to every prompt, word for word:**
> *"— flat-shaded low-poly style 2D game sprite, three-quarter top-down view
> (seen from above and slightly in front, like a 3/4 overhead RPG camera, NOT
> 45-degree diamond isometric), simple geometric facets, warm muted storybook
> palette, thin dark outline around the silhouette, soft flat shading with
> light from the upper left, no texture noise, no pixel art, clean simple
> silhouette, single object centered, plain solid magenta background, no
> ground shadow, no text"*
>
> **Delivery specs:** PNG, ~512×512 (bigger fine — we downscale), ONE object
> per image, same camera + light direction across the whole set. Solid magenta
> (#FF00FF) background beats checkerboard for keying (Nano Banana bakes
> checkerboards with no alpha — known gotcha). True-alpha PNG is even better if
> FLORA offers it. Folder-per-asset like the SFX flow: `spr-tree-green/` etc.,
> numbered takes inside.
>
> **What stays code-drawn (don't generate):** the islands/terrain themselves
> (procedurally shaped — every island unique), piers (they rotate to any
> coast angle), flags/banners/pennants (animated + faction-recolored in code),
> and all ships/monsters (they rotate — different pipeline).

## A — Flora (one per biome; ×2–3 silhouette variants each)

1. **spr-tree-green** ×3 — A single leafy temperate tree with a round faceted canopy in warm mossy greens and a short brown trunk
2. **spr-tree-tropic** ×2 — A single tropical palm tree, slender curved trunk, faceted fan of green fronds
3. **spr-tree-dead** ×2 — A single bare dead tree, gnarled twisted branches, grey-violet ashen wood, ominous
4. **spr-tree-snow** ×2 — A single snow-dusted pine tree, dark blue-green layered branches with white snow caps
5. **spr-cactus** ×2 — A single tall desert saguaro cactus with two arms, dusty sage green with vertical ridges
6. **spr-shrub-green** ×2 *(new flavor, optional)* — A small low round bush in mossy green

## B — Rock & terrain props

7. **spr-rock-stack** ×3 — A single tall weathered sea-stack rock spire, layered grey-brown stone facets
8. **spr-peak-crag** ×2 — A single rocky mountain crag peak, sharp faceted grey stone with warm brown base
9. **spr-boulder** ×2 *(optional)* — A single rounded granite boulder, mossy on top

## C — Port buildings (the four culture kits — drawPortHouse)

10. **spr-house-euro** — A single small stone fisherman's cottage with a red gabled roof, tiny chimney, one warm lit window
11. **spr-house-norse** — A single small norse stave hall, dark timber planks, steep roof with crossed carved gable beams
12. **spr-runestone** — A single standing norse rune stone with faint glowing cyan runes carved into grey rock
13. **spr-house-east** — A single small two-tier pagoda with upturned eaves and a gold finial on top
14. **spr-house-desert** — A single small adobe dome house in warm sand tones with a rounded doorway
15. **spr-obelisk** — A single slender sandstone obelisk with a gilt gold tip

## D — Forts & strongholds (2 states each — the banner stays code-drawn on top)

16. **spr-fort** — A single small stone fortress keep: square curtain walls with crenellations, one round tower, gun ports, weathered grey stone with warm brown timber details
17. **spr-fort-razed** — The same small stone fortress in ruins: collapsed walls, rubble heaps, charred timbers, faint smoke stains

## E — Water-side props

18. **spr-flotsam** ×2 — A single piece of floating shipwreck flotsam: a broken crate and plank tangle, waterlogged wood
19. **spr-crate** / **spr-barrel** ×2 each *(optional — loot bobbing in water)* — A single wooden cargo crate with rope binding / A single oak barrel with iron hoops
20. **spr-iceberg** ×2 — A single drifting iceberg, faceted pale blue-white ice with a crown ridge

## F — Palette anchors (steer generations toward the game's colors)

- Temperate greens: mossy `#6fa05a` → deep `#4a7a44` · trunks `#7a5c34`
- Tropic: brighter palm green `#5fae6a`, sand `#e4c686`
- Dead/cursed: ash grey-violet `#7a6e86`, bone `#a89a8c`
- Snow: pine `#3e5a52`, snow `#e8f0f4` with blue shadow `#b8ccd8`
- Desert: sand `#e4c686`, adobe `#d8b07c`, cactus sage `#8aa86a`
- Stone: warm grey `#8a8478`, cliffs `#98764e`
- Wood: teak `#7a5230`, weathered `#9a7c54`
- Always: thin near-black outline `#1a1410`-ish, light from upper left

## G — Integration plan (for the wiring session, when assets land)

- Pipeline: the existing `drawSprite` flat-sprite path (SPRITES/SPR loader) —
  correct for all non-rotating assets. Key decision per HANDOFF: land props
  likely draw INTO the pixel buffer (bctx) so they pixelate with the terrain,
  vs. the crisp overlay used for the ghost ship. Decide by eye with the first
  tree.
- Variants pick deterministically by the object's world hash (same tree every
  visit). Scale by existing per-object `s` factors; y-sort keys unchanged.
- Known wrinkle (HANDOFF §6): depth-sorting crisp sprites vs code-drawn ships.
- Process each delivery: key magenta → alpha (PIL, same throwaway-script flow
  as the ghost ship), downscale, drop in `assets/sprites/`, wire one draw
  function at a time (drawTree → drawPortHouse → fort → props), verify per
  biome + culture, commit per batch.
