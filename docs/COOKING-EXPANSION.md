# CookingExpansion — Skill-Based Cooking for Slimefun (Paper 26.2 + Folia)

A Slimefun addon that adds **real, skill-based cooking** — not passive "insert → wait → collect"
machines. Three cooking stations each run a distinct real-time minigame; how well you play decides
the **quality tier** of the dish, and the tier drives its **food value + potion buffs**.

- **Target:** Paper API `26.2` and **Folia** (verified on SourbyCraft / Luminol 26.2)
- **Depends on:** Slimefun v5.1 fork (this hub) — consumes the relocated SuDo library it ships
- **Java:** 25 · **Item text:** MiniMessage · **Folia-safe:** every minigame ticks on the player's
  region via the SuDo scheduler; nothing touches a foreign region

---

## The core idea — quality tiers

Every dish comes out at one of three tiers, decided by your performance in that station's minigame:

| Tier | Food | Buffs |
|---|---|---|
| ⭐ **Perfect** | full hunger + high saturation | full buff set (3 min) |
| **Normal** | reduced hunger | reduced buff (1 min) |
| 🔥 **Burnt** | 2 hunger | 10 s Hunger penalty |

The tier is stored on the dish (PDC) and applied when eaten — nutrition/saturation come from the
item's food component, buffs from the eat listener.

---

## Machine 1 — Stove (heat + flip)

Load ingredients, click **Cook**, then juggle two things at once for the cook's duration:

- **Heat management** — keep the gauge inside the recipe's target zone with the ▲/▼ buttons. Heat
  drifts on its own and the drift direction flips at random, so you must watch, not spam. Sit at
  ≥ 95° for 40 ticks and the dish instantly **burns**.
- **Flip cues** — when **⭐ FLIP!** flashes (title + bell), click within 1.5 s. Hits score big,
  misses and spam cost you.

Your score across both → the tier.

### Stove recipes (8)

| Dish | Ingredients | Perfect (3 min) | Normal (1 min) |
|---|---|---|---|
| Seared Steak | beef + milk bucket | Strength I | — |
| Golden Fried Fish | salmon + egg + wheat | Water Breathing + Haste I | Haste I |
| Hearty Stew | beef + potato + carrot + bowl | Regen I + Absorption I | Absorption I |
| Miner's Hotpot | rabbit + baked potato + red mushroom + bowl | Haste II + Night Vision | Haste I |
| Blazing Curry | chicken + blaze powder + wheat + bowl | Fire Resistance + Strength I | Fire Resistance 30 s |
| Traveler's Skewer | mutton + carrot + stick | Speed II | Speed I |
| Sweet Berry Tart | sweet berries + egg + wheat + sugar | Regen II 30 s + Absorption II | Regen I 30 s |
| Chef's Feast | cooked beef + cooked salmon + baked potato + carrot + bowl | Saturation + Strength I + Speed I | Saturation 30 s |

Milk buckets return an empty bucket on a successful cook.

---

## Machine 2 — Oven (follow the bake curve)

A different minigame: instead of holding a fixed zone, you **track a target temperature that moves**
through phases — **Preheat → Bake → Rest**. Nudge ▲/▼ to keep your temp on the target band as it
shifts; a title + sound fires on every phase change. The gauge marker glows **gold** when you're on
target, **red** when you drift. Over-bake (≥ 95° for 40 ticks) and it **scorches** to Burnt.

### Oven recipes (5)

| Dish | Ingredients | Perfect (3 min) | Normal (1 min) |
|---|---|---|---|
| Artisan Bread | wheat ×3 | Saturation + Speed I | Saturation 30 s |
| Pumpkin Pie Supreme | pumpkin + sugar + egg | Regeneration I + Absorption I | Regeneration I |
| Sweet Cake | wheat + sugar + egg + milk bucket | Absorption II + Jump Boost I | Absorption I |
| Honey Cookies | wheat + honey bottle + cocoa beans | Haste I + Speed I | Haste I |
| Glow-Berry Tart | glow berries + sugar + wheat + egg | Night Vision + Regeneration I | Night Vision |

Milk buckets → empty bucket; honey bottles → empty glass bottle, on success.

---

## Machine 3 — Prep Station (chop combo)

Processes raw ingredients into **5 prepped goods** via a quick, forgiving chop-combo: hit the **CHOP**
button on each beat. A clean combo yields more (flawless → +1 bonus); a sloppy one yields fewer — but
**never zero**, and there is **no fail/burn state**. Prepped goods themselves are plain intermediate
items (no tier, not edible) — their value is unlocking deeper recipes.

### Prepped goods (5)

| Prepped good | Made from |
|---|---|
| Sliced Meat | beef |
| Chopped Veggies | carrot + potato |
| Flour | wheat |
| Ground Spice | blaze powder |
| **Dough** | **Flour** (prepped) + egg — a 2-step chain: wheat → flour → dough |

### Prepped-input dishes (6)

These require prepped goods (raw substitutes are rejected):

| Dish | Machine | Ingredients | Perfect (3 min) |
|---|---|---|---|
| Gourmet Stir-fry | Stove | Sliced Meat + Chopped Veggies + Ground Spice | Strength I + Speed I |
| Spiced Kebab | Stove | Sliced Meat + Ground Spice + stick | Strength II |
| Hearty Dumplings | Stove | Dough + Chopped Veggies + bowl | Absorption II + Saturation |
| Stuffed Bun | Oven | Dough + Sliced Meat | Saturation + Strength I |
| Veggie Tart | Oven | Dough + Chopped Veggies + egg | Regen I + Absorption I |
| Spice Cake | Oven | Flour + Ground Spice + sugar + egg | Haste II + Fire Resistance |

---

## How ingredient matching works

Recipe ingredients are an **`Ingredient`** type — either a vanilla `Material` **or** a prepped
Slimefun item (by id). A single resolver (`Ingredients.fromStack`) classifies a slot as *prepped*
only if it is one of the 5 real prepped goods (closed-set membership — not a name prefix), so a
prepped item never accidentally matches its raw base material, and no other item leaks in.

## Crafting the machines

All three are craftable in the Slimefun guide under the **Cooking** category:

- **Stove** — furnace + iron + campfire pattern
- **Oven** — furnace + bricks + iron
- **Prep Station** — crafting table + iron + iron axe

## Folia safety

Each station has a pure, headless minigame state machine (`CookingSession` / `BakingSession` /
`PrepSession`) ticked once per tick via `Scheduler.runAtEntityTimer(player, …)` — the player's
region thread. Sessions are ephemeral; every end path (GUI close, quit, death, block broken, server
shutdown) funnels through one idempotent `abort()` gated on an atomic map removal, and refunds drop
at the **block location** (survives an offline mid-cook quit). Holograms are marshalled through the
block/entity schedulers. No dish-dup, no item loss, no wrong-region access.

## Content summary

**27 items** — 3 machines + 8 stove dishes + 5 oven dishes + 5 prepped goods + 6 prepped-input dishes.

---

*Part of the [SF-ISSUE](https://github.com/YanIanZ/SF-ISSUE) Slimefun v5.1 fork docs hub. Requires
the fork build (Paper 26.2 / Folia). Declare `folia-supported: true` in any addon's plugin.yml or
Folia refuses to load it.*
