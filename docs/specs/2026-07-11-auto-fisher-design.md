# Auto-Fisher — Design Spec

**Date:** 2026-07-11
**Branch:** `feat/sf4c-auto-fisher`
**Status:** Approved

## Summary

A new END_GAME electric machine that auto-fishes while placed touching water,
consuming energy. It rolls a curated weighted loot table (fish / treasure /
junk) and pushes catches into its output slots. An optional bait input (raw
cod / salmon) doubles fishing speed and shifts the odds toward treasure. It
sits in the same machine family as Auto-Breeder and the Exp-Collector.

This is the first sub-project of "SF4c". It produces working, testable
software on its own.

## Global Constraints

Copied verbatim; every task inherits these.

- **Colors:** 100% hex MiniMessage. Every name/lore string uses `<#RRGGBB>`
  tags. No `org.bukkit.ChatColor`, no legacy `§`/`&` codes in source.
- **Item stacks:** Build display items via `dev.yanianz.sudo.items.CustomItemStack`
  (`CustomItemStack.create(Material, name, lore...)` — a bridged sink that
  MiniMessage-renders name + lore). Menu decoration items go through it too.
- **Folia-safe:** The `BlockTicker` MUST return `isSynchronized() == true`.
  All world reads (water check) and inventory writes (`pushItem`) happen on
  the region thread that owns the block. No async world access.
- **SuDo + Paper API only.** No new third-party deps.
- **Energy API:** implement `EnergyNetComponent` (`CONSUMER`), use
  `getCharge(Location)` / `removeCharge(Location, int)`.

## Architecture

`AutoFisher extends SlimefunItem implements InventoryBlock, EnergyNetComponent`
— mirrors `AbstractGrowthAccelerator`: a chest menu (`BlockMenuPreset`), an
energy buffer, a synchronized `BlockTicker`, and a `BlockBreakHandler` that
drops the machine's contents.

Loot rolling is extracted into a pure helper `FishingLoot` so it can be
unit-tested deterministically with a seeded `Random` — the machine itself
only needs a light "does not throw / drains energy correctly" MockBukkit test.

### Files

- **Create** `src/main/java/io/github/thebusybiscuit/slimefun4/implementation/items/electric/machines/FishingLoot.java`
  — pure weighted-loot roller. No Bukkit world/plugin state.
- **Create** `src/main/java/io/github/thebusybiscuit/slimefun4/implementation/items/electric/machines/AutoFisher.java`
  — the machine (menu, energy, water check, tick, bait handling).
- **Modify** `src/main/java/io/github/thebusybiscuit/slimefun4/implementation/SlimefunItems.java`
  — add `AUTO_FISHER` `SlimefunItemStack` field near the other machines.
- **Modify** `src/main/java/io/github/thebusybiscuit/slimefun4/implementation/setup/SlimefunItemSetup.java`
  — register `AutoFisher` with its recipe, near the accelerators (~line 2375).
- **Modify** `src/main/java/io/github/thebusybiscuit/slimefun4/implementation/setup/ResearchSetup.java`
  — research gate `register("auto_fisher", 282, ...)`.
- **Create** `src/test/java/io/github/thebusybiscuit/slimefun4/implementation/items/electric/machines/TestFishingLoot.java`
- **Create** `src/test/java/io/github/thebusybiscuit/slimefun4/implementation/items/electric/machines/TestAutoFisher.java`

## Component: FishingLoot

Pure roller. No Bukkit singletons.

```java
public final class FishingLoot {
    // A weighted entry: a prototype ItemStack + a weight.
    // roll picks one entry proportional to weight, returns a fresh clone.
    public static ItemStack roll(Random random, boolean baited);
}
```

- Two internal weighted pools selected by `baited`:
  - **Unbaited** — Common 85 (COD 30, SALMON 25, TROPICAL_FISH 17,
    PUFFERFISH 13), Treasure 5 (ENCHANTED_BOOK, NAUTILUS_SHELL, NAME_TAG,
    SADDLE, BOW — weight 1 each), Junk 10 (LEATHER_BOOTS, STICK, BOWL,
    ROTTEN_FLESH, LILY_PAD — weight 2 each).
  - **Baited** — same entries, re-weighted so Treasure total ≈ 12 and Junk
    total ≈ 4 (Common ≈ 84). Concretely: Treasure weight 2.4→ use integer
    weights ×5 of the unbaited pool then bump treasure entries to weight 12
    total / junk to 4 total. Implementation: keep the two pools as explicit
    static arrays with integer weights; do NOT compute at runtime.
- `roll` returns a **new `ItemStack`** each call (clone of the prototype),
  never the shared prototype, so callers can mutate freely.
- Deterministic: same `Random` seed + same `baited` → same sequence.

## Component: AutoFisher

### Menu (27-slot chest)

- **Bait input slots:** `{ 10, 11 }`
- **Output slots:** `{ 14, 15, 16, 23, 24, 25 }`
- **Border:** every other slot (`0-9, 12, 13, 17, 18, 19, 20, 21, 22, 26`),
  filled with `CustomItemStack.create(Material.CYAN_STAINED_GLASS_PANE, " ")`
  + `ChestMenuUtils.getEmptyClickHandler()`.
- `getInputSlots()` → `{10, 11}`; `getOutputSlots()` → `{14,15,16,23,24,25}`.
- `BlockBreakHandler` (SimpleBlockBreakHandler): on break, drop input +
  output slots (`inv.dropItems(loc, getInputSlots())` and output slots).

### Energy

- `getEnergyComponentType()` → `EnergyNetComponentType.CONSUMER`.
- `getCapacity()` → `512`.
- `ENERGY_PER_CATCH = 30` (J per catch).

### Placement / water check

- `private boolean hasWater(Block b)` — true if the block **below** OR any of
  the 4 horizontal neighbors (`b.getRelative(face)` for NORTH/EAST/SOUTH/WEST)
  is water. "Is water" = `Material.WATER` or a waterlogged block
  (`BlockData instanceof Waterlogged && ((Waterlogged) data).isWaterlogged()`).
- No water touching → tick returns immediately, **no energy consumed**.

### Ticking (synchronized `BlockTicker`)

Per tick (`tick(Block b)`):

1. If `getCharge(loc) < ENERGY_PER_CATCH` → return (idle).
2. If `!hasWater(b)` → return (idle, no drain).
3. Read progress counter from BlockStorage:
   `int progress = Integer.parseInt(BlockStorage.getLocationInfo(loc, "fishing_progress"))`
   defaulting to `0` when the value is `null`/unparseable.
4. Determine interval: `baited = firstBaitSlot(inv) != -1`;
   `int needed = baited ? 3 : 6`.
5. If `progress + 1 < needed` → `addBlockInfo(loc, "fishing_progress", ""+(progress+1))`, return.
6. Otherwise a catch fires:
   - `ItemStack catch = FishingLoot.roll(ThreadLocalRandom.current(), baited)`.
   - `ItemStack leftover = inv.pushItem(catch, getOutputSlots())`.
   - If `leftover != null` (output full): do **not** drain energy, do **not**
     reset progress, do **not** consume bait — return (hold state).
   - Else: `removeCharge(loc, ENERGY_PER_CATCH)`; if `baited` consume 1 from
     the bait slot (`inv.consumeItem(baitSlot)`); reset progress to `0`.

- `int firstBaitSlot(BlockMenu inv)` — returns the first input slot holding a
  bait item, else -1. Bait = raw `Material.COD` or `Material.SALMON` (any
  non-Slimefun stack of that material). Use `SlimefunItem.getByItem(...) == null`
  guard so a Slimefun item that happens to be a cod isn't eaten.

### Registration (SlimefunItemSetup)

```java
new AutoFisher(itemGroups.electricity, SlimefunItems.AUTO_FISHER, RecipeType.ENHANCED_CRAFTING_TABLE,
    new ItemStack[] {
        new ItemStack(Material.FISHING_ROD), new ItemStack(Material.PRISMARINE_SHARD), new ItemStack(Material.FISHING_ROD),
        SlimefunItems.ELECTRIC_MOTOR.item(), new ItemStack(Material.WATER_BUCKET), SlimefunItems.ELECTRIC_MOTOR.item(),
        SlimefunItems.REINFORCED_ALLOY_INGOT.item(), SlimefunItems.BIG_CAPACITOR.item(), SlimefunItems.REINFORCED_ALLOY_INGOT.item()
    }).register(plugin);
```

### Item definition (SlimefunItems)

```java
public static final SlimefunItemStack AUTO_FISHER = new SlimefunItemStack("AUTO_FISHER", Material.PRISMARINE,
    "<#48CAE4>Auto-Fisher", "",
    "<#F1FAEE>Auto-fishes while touching water", "",
    LoreBuilder.machine(MachineTier.END_GAME, MachineType.MACHINE),
    LoreBuilder.powerBuffer(512),
    "<#4A4E69>⇨ <#8D99AE>Bait (raw cod/salmon) doubles speed & luck");
```

### Research (ResearchSetup)

```java
register("auto_fisher", 282, "The Art of Automated Angling", 25, SlimefunItems.AUTO_FISHER);
```

## Data flow

```
tick → charge ok? → water? → progress++ … at threshold →
  FishingLoot.roll → pushItem(output) →
    full?  → hold (no drain/consume/reset)
    ok     → removeCharge · consume bait · reset progress
```

## Error handling / edge cases

- Missing/garbage `fishing_progress` value → treated as 0 (parse guarded).
- Output full → machine idles without wasting energy or bait (state held).
- Slimefun item shaped like a cod placed as "bait" → not consumed (guard).
- No water → no drain (prevents griefing energy grids in dry biomes).
- Folia: sync ticker guarantees region-thread ownership for all world/inv ops.

## Testing

**TestFishingLoot** (no MockBukkit needed — pure):
- Seeded `Random` → `roll` returns a non-null valid ItemStack.
- Over N=10_000 rolls, every returned material is in the declared pool.
- Baited pool returns strictly more treasure-category items than unbaited
  over the same N with the same seed sequence.
- `roll` returns a fresh instance (mutating the result doesn't affect the next
  roll of the same material).

**TestAutoFisher** (MockBukkit):
- Machine registers without throwing; `getById("AUTO_FISHER")` non-null.
- `getInputSlots()`/`getOutputSlots()` return the specified arrays.
- Water check: block with WATER below → true; dry surroundings → false.
- Registration wiring: item is in the `electricity` group.

## Out of scope (YAGNI)

- No second machine tier (single END_GAME machine).
- No custom bait item (reuse vanilla raw cod/salmon).
- No vanilla `LootTables.FISHING` integration (curated table chosen for
  determinism + testability).
- No fishing animation / bobber entity.
