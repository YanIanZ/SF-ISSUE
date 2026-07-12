# Mystery Box — Design Spec

**Date:** 2026-07-12
**Branch:** `feat/sf4d-mystery-box`
**Status:** Approved

## Summary

A placed, non-electric magical Slimefun block. The player opens it, drops a
**Mystery Key** into the input slot, and clicks the **Spin** button. The block
consumes one key, rolls a curated weighted reward table (common vanilla
resources / rare Slimefun items / a rare jackpot), pushes the reward into the
output slot, and plays a firework + sound. Jackpots broadcast to the server.

Loot rolling is a pure, seed-testable `MysteryLoot` helper (same shape as the
existing `FishingLoot`). The block is event-driven (a menu click handler) — no
ticker, no energy.

## Global Constraints

Copied verbatim; every task inherits these.

- **Colors:** 100% hex MiniMessage — every name/lore string uses `<#RRGGBB>`
  tags. No `org.bukkit.ChatColor`, no legacy `§`/`&` codes in source.
- **Item stacks / display items:** build via
  `dev.yanianz.sudo.items.CustomItemStack.create(Material, name, lore...)`
  (bridged sink, MiniMessage-renders name + lore). Menu decoration items too.
- **Server broadcast / messages:** render via `dev.yanianz.sudo.common.MiniMessages`
  (`MiniMessages.component(String)`), never a raw String sink.
- **Folia-safe:** the Spin click handler runs on the block's region thread
  (menu interaction). The firework spawn goes through `FireworkUtils`
  (already region-scheduled). `Bukkit.broadcast(Component)` is thread-safe.
  No async world access.
- **SuDo + Paper API only.** No new third-party dependencies.
- **Non-electric:** the block does NOT implement `EnergyNetComponent` and has
  no `BlockTicker`.

## Architecture

Two production classes + item/registration/research wiring:

- **`MysteryLoot`** — pure weighted-loot roller. `roll(Random)` returns a
  `MysteryReward(ItemStack item, boolean jackpot)`. Deterministic for a given
  `Random` sequence. Three static weighted pools (common / rare / jackpot).
- **`MysteryBox`** — `extends SlimefunItem implements InventoryBlock`. Builds a
  custom `BlockMenuPreset` (like `ReactorAccessPort`) whose `newInstance`
  attaches a block-aware Spin click handler. A `BlockBreakHandler` drops the
  block's input + output contents.

### Files

- **Create** `implementation/items/blocks/MysteryLoot.java`
- **Create** `implementation/items/blocks/MysteryBox.java`
- **Modify** `implementation/SlimefunItems.java` — add `MYSTERY_BOX` + `MYSTERY_KEY`.
- **Modify** `implementation/setup/SlimefunItemSetup.java` — register both, with recipes.
- **Modify** `implementation/setup/ResearchSetup.java` — research `mystery_box` id 283.
- **Modify** `languages/en/researches.yml` — `mystery_box` translation.
- **Create** `test/.../implementation/items/blocks/TestMysteryLoot.java`
- **Create** `test/.../implementation/items/blocks/TestMysteryBox.java`

## Component: MysteryReward

A small immutable result returned by `MysteryLoot.roll`.

```java
public record MysteryReward(ItemStack item, boolean jackpot) {}
```

- `item` — a fresh, non-null `ItemStack` (never a shared prototype).
- `jackpot` — true only when the roll came from the jackpot pool (drives the
  broadcast + extra firework).

## Component: MysteryLoot

Pure roller. Pools are explicit static arrays of `(int weight, Rarity, Supplier<ItemStack>)`.

```java
public final class MysteryLoot {
    public static MysteryReward roll(Random random);
}
```

- **Common** (total weight 70) — vanilla, fixed amounts:
  IRON_INGOT x6 (w15), GOLD_INGOT x4 (w13), REDSTONE x8 (w10), LAPIS_LAZULI x6
  (w10), COAL x8 (w10), EXPERIENCE_BOTTLE x4 (w7), DIAMOND x1 (w3),
  EMERALD x2 (w2). Sum = 70.
- **Rare** (total weight 25) — Slimefun items via `SlimefunItems.X.item().clone()`:
  GOLD_24K (w5), REINFORCED_ALLOY_INGOT (w4), SYNTHETIC_DIAMOND (w4),
  SYNTHETIC_EMERALD (w3), MAGIC_LUMP_2 (w3), DAMASCUS_STEEL_INGOT (w3),
  GILDED_IRON (w3). Sum = 25.
- **Jackpot** (total weight 5) — Slimefun items, `jackpot = true`:
  CARBONADO (w2), MAGIC_LUMP_3 (w2), REINFORCED_ALLOY_INGOT x4 (w1). Sum = 5.

- Total weight 100. `roll` picks proportional to weight across all three pools
  concatenated, evaluates the entry's `Supplier<ItemStack>` (returning a fresh
  clone), and returns `new MysteryReward(item, rarity == JACKPOT)`.
- Deterministic: same seed → same sequence.
- Suppliers are evaluated at roll time (not class-init) so Slimefun item stacks
  resolve correctly. Vanilla entries use `new ItemStack(Material, amount)`.

## Component: MysteryBox

### Menu (27-slot chest)

- **Input (key) slot:** `11`
- **Spin button:** `13` (a `CustomItemStack` — `Material.NETHER_STAR`,
  `<#C77DFF>✦ Spin` + lore). Updated/attached in `newInstance`.
- **Output slot:** `15`
- **Border:** every other slot, `ChestMenuUtils.getBackground()` +
  `getEmptyClickHandler()`; slot 11 uses `getInputSlotTexture()` as its
  background hint and slot 15 uses `getOutputSlotTexture()`.
- `getInputSlots()` → `{11}`; `getOutputSlots()` → `{15}`.

### Preset (custom BlockMenuPreset, modelled on ReactorAccessPort)

```java
new BlockMenuPreset(getId(), "<#C77DFF>Mystery Box") {
    public void init() { constructMenu(this); }
    public boolean canOpen(Block b, Player p) {
        return p.hasPermission("slimefun.inventory.bypass")
            || Slimefun.instance().isUnitTest()
            || Slimefun.getProtectionManager().hasPermission(p, b.getLocation(), Interaction.INTERACT_BLOCK);
    }
    public void newInstance(BlockMenu menu, Block b) {
        menu.addMenuClickHandler(SPIN_SLOT, (p, slot, item, action) -> { spin(p, b, menu); return false; });
    }
    public int[] getSlotsAccessedByItemTransport(ItemTransportFlow flow) {
        return flow == ItemTransportFlow.INSERT ? getInputSlots() : getOutputSlots();
    }
};
```

- `constructMenu` draws the border, the input/output textures, and the Spin
  button item; sets `getDefaultOutputHandler()` on slot 15 so players can
  withdraw the reward; `getEmptyClickHandler()` on border + spin slot (the
  block-aware spin handler is attached per-instance in `newInstance`).

### Spin logic (`spin(Player p, Block b, BlockMenu menu)`)

1. `ItemStack key = menu.getItemInSlot(11)`. If `!isMysteryKey(key)` → send
   `p` the "insert a key" message, return.
2. `MysteryReward reward = MysteryLoot.roll(ThreadLocalRandom.current())`.
3. `ItemStack leftover = menu.pushItem(reward.item(), getOutputSlots())`.
   If `leftover != null` (output full) → send "output full" message, return
   (do NOT consume the key).
4. `menu.consumeItem(11, 1)`.
5. Effects: `FireworkUtils.launchFirework(b.getLocation().add(0.5, 1, 0.5), color)`;
   `b.getWorld().playSound(b.getLocation(), Sound.BLOCK_NOTE_BLOCK_PLING, 1f, pitch)`.
6. If `reward.jackpot()` → `b.getWorld().playSound(b.getLocation(), Sound.UI_TOAST_CHALLENGE_COMPLETE, 1f, 1f)`;
   a second firework; broadcast
   `MiniMessages.component("<#E9C46A>✦ " + p.getName() + " hit the <#C77DFF>Mystery Box <#E9C46A>jackpot!")`.

- `isMysteryKey(ItemStack)` → `SlimefunUtils.isItemSimilar(item, SlimefunItems.MYSTERY_KEY, true)`.

### Break handler

`SimpleBlockBreakHandler` → on break, `inv.dropItems(loc, getInputSlots())` and
`inv.dropItems(loc, getOutputSlots())`.

### Item definitions (SlimefunItems)

```java
public static final SlimefunItemStack MYSTERY_BOX = new SlimefunItemStack("MYSTERY_BOX", Material.ENDER_CHEST, "<#C77DFF>Mystery Box", "", "<#F1FAEE>Insert a <#C77DFF>Mystery Key <#F1FAEE>and spin", "<#F1FAEE>for a random reward!", "", "<#E9C46A>✦ Jackpot chance!");
public static final SlimefunItemStack MYSTERY_KEY = new SlimefunItemStack("MYSTERY_KEY", Material.TRIPWIRE_HOOK, "<#C77DFF>Mystery Key", "", "<#F1FAEE>Spin a <#C77DFF>Mystery Box <#F1FAEE>with this");
```

### Registration (SlimefunItemSetup, itemGroups.magicalGadgets)

```java
new MysteryBox(itemGroups.magicalGadgets, SlimefunItems.MYSTERY_BOX, RecipeType.ENHANCED_CRAFTING_TABLE,
    new ItemStack[] {
        SlimefunItems.GOLD_24K.item(), new ItemStack(Material.DIAMOND), SlimefunItems.GOLD_24K.item(),
        new ItemStack(Material.DIAMOND), new ItemStack(Material.ENDER_CHEST), new ItemStack(Material.DIAMOND),
        SlimefunItems.GOLD_24K.item(), new ItemStack(Material.DIAMOND), SlimefunItems.GOLD_24K.item()
    }).register(plugin);

SlimefunItems.MYSTERY_KEY.item(); // MYSTERY_KEY registered as an UnplaceableBlock-style simple item:
new SlimefunItem(itemGroups.magicalGadgets, SlimefunItems.MYSTERY_KEY, RecipeType.ENHANCED_CRAFTING_TABLE,
    new ItemStack[] {
        null, new ItemStack(Material.GOLD_INGOT), null,
        new ItemStack(Material.GOLD_INGOT), new ItemStack(Material.TRIPWIRE_HOOK), new ItemStack(Material.GOLD_INGOT),
        null, new ItemStack(Material.GOLD_INGOT), null
    }, new SlimefunItemStack(SlimefunItems.MYSTERY_KEY, 2)).register(plugin);
```

(The key recipe yields 2 keys per craft. If the `SlimefunItem` output-amount
constructor differs, use the standard `RecipeType`/output pattern already used
by other simple craftable items — the intent is: a plain craftable item, no
custom behaviour.)

### Research (ResearchSetup) + translation

```java
register("mystery_box", 283, "A Box of Mysteries", 25, SlimefunItems.MYSTERY_BOX, SlimefunItems.MYSTERY_KEY);
```
`researches.yml`: `mystery_box: A Box of Mysteries`.

## Data flow

```
click Spin → key present? → roll MysteryLoot → pushItem(output) →
  full?  → message, keep key
  ok     → consumeItem(key) · firework · sound · (jackpot? broadcast + 2nd firework)
```

## Error handling / edge cases

- No key in input → message, no roll.
- Output full → message, key NOT consumed (no reward lost).
- Slimefun item shaped like the key material → identity check via
  `SlimefunUtils.isItemSimilar(..., true)` (compare-lore true) prevents a
  vanilla tripwire hook from counting as a key.
- Folia: click handler on region thread; firework region-scheduled; broadcast
  thread-safe.

## Testing

**TestMysteryLoot** (MockBukkit — rewards build real ItemStacks):
- Seeded `Random` → `roll` returns a non-null item every time over N=10_000.
- Over N rolls, `jackpot`-count < rare-count < common-count (weights 5<25<70).
- `roll` returns a fresh instance (mutating the result's amount doesn't affect
  the next identical roll).
- Every rolled item's type is within the declared union of pool materials
  (vanilla materials + the Slimefun items' materials).

**TestMysteryBox** (MockBukkit):
- Registers without throwing; `getById("MYSTERY_BOX")` non-null.
- `getInputSlots()` == `{11}`, `getOutputSlots()` == `{15}`.
- `isMysteryKey` (package-visible) true for a `MYSTERY_KEY` stack, false for a
  bare `TRIPWIRE_HOOK`.

## Out of scope (YAGNI)

- No energy / ticker.
- No cost beyond the key (no gambling economy).
- No multi-tier boxes, no configurable tables (fixed curated table).
- No animation beyond the firework + sound.
