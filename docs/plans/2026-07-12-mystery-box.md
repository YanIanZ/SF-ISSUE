# Mystery Box Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax.

**Goal:** Add a non-electric "Mystery Box" Slimefun block that consumes a Mystery Key and spins a weighted random reward (vanilla common / Slimefun rare / jackpot) with firework + broadcast.

**Architecture:** `MysteryLoot` is a pure seed-testable weighted roller returning a `MysteryReward(item, jackpot)`. `MysteryBox extends SlimefunItem implements InventoryBlock` with a custom `BlockMenuPreset` (modelled on `ReactorAccessPort`) whose `newInstance` attaches a block-aware Spin click handler; a `BlockBreakHandler` drops contents. Items + recipes registered in `SlimefunItemSetup`, gated by a research.

**Tech Stack:** Java, Slimefun4 API, SuDo (`CustomItemStack`, `MiniMessages`, scheduler-backed `FireworkUtils`), Paper API, MockBukkit + JUnit 5, Gradle.

## Global Constraints

- **Colors:** 100% hex MiniMessage (`<#RRGGBB>`). No `org.bukkit.ChatColor`, no legacy `§`/`&` in source.
- **Display items:** `dev.yanianz.sudo.items.CustomItemStack.create(Material, name, lore...)`.
- **Broadcast/messages:** `dev.yanianz.sudo.common.MiniMessages.component(String)` — never a raw String sink.
- **Folia-safe:** Spin handler runs on the block's region thread; firework via `FireworkUtils` (already region-scheduled); `Bukkit.broadcast(Component)` is thread-safe. No async world access.
- **Non-electric:** no `EnergyNetComponent`, no `BlockTicker`.
- **SuDo + Paper API only.** No new deps.
- **Build/test:** `./gradlew test` from `/Users/rheninxy/Client-Project/Slimefun4`. Trust Gradle, not the LSP. Never stage `.gradle/`.
- **Commits:** end every message with `Claude-Session: https://claude.ai/code/session_013YFwH85eQ1mJXua7Hqcu1k`. Stage only the files each task names.

---

### Task 1: MysteryReward + MysteryLoot roller

**Files:**
- Create: `src/main/java/io/github/thebusybiscuit/slimefun4/implementation/items/blocks/MysteryLoot.java`
- Test: `src/test/java/io/github/thebusybiscuit/slimefun4/implementation/items/blocks/TestMysteryLoot.java`

**Interfaces:**
- Produces: `MysteryLoot.roll(Random)` → `MysteryLoot.MysteryReward` (nested record `MysteryReward(ItemStack item, boolean jackpot)`); `MysteryLoot.isJackpotMaterial(Material)` and `MysteryLoot.poolMaterials()` (a `Set<Material>` of every material any pool can produce) for tests.

- [ ] **Step 1: Write the failing test**

```java
package io.github.thebusybiscuit.slimefun4.implementation.items.blocks;

import java.util.Random;

import org.bukkit.inventory.ItemStack;
import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.Assertions;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import org.mockbukkit.mockbukkit.MockBukkit;

class TestMysteryLoot {

    // roll builds real ItemStacks (incl. Slimefun items) — needs the server + plugin loaded.
    @BeforeAll
    public static void load() {
        MockBukkit.mock();
        MockBukkit.load(io.github.thebusybiscuit.slimefun4.implementation.Slimefun.class);
    }

    @AfterAll
    public static void unload() {
        MockBukkit.unmock();
    }

    @Test
    @DisplayName("roll returns a non-null item from a known pool material")
    void testRollInPool() {
        Random r = new Random(1234L);
        for (int i = 0; i < 10_000; i++) {
            MysteryLoot.MysteryReward reward = MysteryLoot.roll(r);
            Assertions.assertNotNull(reward);
            Assertions.assertNotNull(reward.item());
            Assertions.assertTrue(MysteryLoot.poolMaterials().contains(reward.item().getType()),
                "unexpected material: " + reward.item().getType());
        }
    }

    @Test
    @DisplayName("rarity distribution: jackpot < rare < common over N rolls")
    void testRarityDistribution() {
        Random r = new Random(7L);
        int jackpot = 0;
        int common = 0;
        for (int i = 0; i < 50_000; i++) {
            MysteryLoot.MysteryReward reward = MysteryLoot.roll(r);
            if (reward.jackpot()) {
                jackpot++;
            }
            // Common materials are the vanilla ones; treat "not jackpot and vanilla" as common-ish.
            if (MysteryLoot.isCommonMaterial(reward.item().getType())) {
                common++;
            }
        }
        Assertions.assertTrue(jackpot > 0, "jackpot should occur at least once over 50k rolls");
        Assertions.assertTrue(common > jackpot, "common (" + common + ") should exceed jackpot (" + jackpot + ")");
    }

    @Test
    @DisplayName("roll returns a fresh instance (mutation-safe)")
    void testFreshInstance() {
        ItemStack a = MysteryLoot.roll(new Random(42L)).item();
        a.setAmount(64);
        ItemStack b = MysteryLoot.roll(new Random(42L)).item();
        Assertions.assertNotEquals(64, b.getAmount(), "roll must return a fresh item, not a shared prototype");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `./gradlew test --tests "io.github.thebusybiscuit.slimefun4.implementation.items.blocks.TestMysteryLoot"`
Expected: FAIL — `MysteryLoot` does not exist.

- [ ] **Step 3: Write the implementation**

```java
package io.github.thebusybiscuit.slimefun4.implementation.items.blocks;

import java.util.HashSet;
import java.util.Random;
import java.util.Set;
import java.util.function.Supplier;

import javax.annotation.Nonnull;

import org.bukkit.Material;
import org.bukkit.inventory.ItemStack;

import io.github.thebusybiscuit.slimefun4.implementation.SlimefunItems;

/**
 * A pure, deterministic weighted loot roller for the {@link MysteryBox}.
 *
 * <p>Three static weighted pools — common (vanilla resources), rare (Slimefun items) and a rare
 * jackpot — total a weight of 100. Rolls are deterministic for a given {@link Random} sequence, so
 * the reward table is unit-testable without a live server. Each roll returns a fresh {@link ItemStack}.
 *
 * @author Yan
 *
 * @see MysteryBox
 */
public final class MysteryLoot {

    private MysteryLoot() {}

    /**
     * The outcome of a single {@link #roll(Random)} — the reward item and whether it was a jackpot.
     */
    public record MysteryReward(@Nonnull ItemStack item, boolean jackpot) {}

    private enum Rarity {
        COMMON,
        RARE,
        JACKPOT
    }

    private static final class Entry {

        private final int weight;
        private final Rarity rarity;
        private final Supplier<ItemStack> supplier;

        private Entry(int weight, Rarity rarity, Supplier<ItemStack> supplier) {
            this.weight = weight;
            this.rarity = rarity;
            this.supplier = supplier;
        }
    }

    // Common 70 (vanilla, fixed amounts): 15+13+10+10+10+7+3+2 = 70.
    // Rare 25 (Slimefun): 5+4+4+3+3+3+3 = 25.
    // Jackpot 5 (Slimefun): 2+2+1 = 5. Total = 100.
    private static final Entry[] POOL = {
        new Entry(15, Rarity.COMMON, () -> new ItemStack(Material.IRON_INGOT, 6)),
        new Entry(13, Rarity.COMMON, () -> new ItemStack(Material.GOLD_INGOT, 4)),
        new Entry(10, Rarity.COMMON, () -> new ItemStack(Material.REDSTONE, 8)),
        new Entry(10, Rarity.COMMON, () -> new ItemStack(Material.LAPIS_LAZULI, 6)),
        new Entry(10, Rarity.COMMON, () -> new ItemStack(Material.COAL, 8)),
        new Entry(7, Rarity.COMMON, () -> new ItemStack(Material.EXPERIENCE_BOTTLE, 4)),
        new Entry(3, Rarity.COMMON, () -> new ItemStack(Material.DIAMOND, 1)),
        new Entry(2, Rarity.COMMON, () -> new ItemStack(Material.EMERALD, 2)),

        new Entry(5, Rarity.RARE, () -> SlimefunItems.GOLD_24K.item().clone()),
        new Entry(4, Rarity.RARE, () -> SlimefunItems.REINFORCED_ALLOY_INGOT.item().clone()),
        new Entry(4, Rarity.RARE, () -> SlimefunItems.SYNTHETIC_DIAMOND.item().clone()),
        new Entry(3, Rarity.RARE, () -> SlimefunItems.SYNTHETIC_EMERALD.item().clone()),
        new Entry(3, Rarity.RARE, () -> SlimefunItems.MAGIC_LUMP_2.item().clone()),
        new Entry(3, Rarity.RARE, () -> SlimefunItems.DAMASCUS_STEEL_INGOT.item().clone()),
        new Entry(3, Rarity.RARE, () -> SlimefunItems.GILDED_IRON.item().clone()),

        new Entry(2, Rarity.JACKPOT, () -> SlimefunItems.CARBONADO.item().clone()),
        new Entry(2, Rarity.JACKPOT, () -> SlimefunItems.MAGIC_LUMP_3.item().clone()),
        new Entry(1, Rarity.JACKPOT, () -> {
            ItemStack stack = SlimefunItems.REINFORCED_ALLOY_INGOT.item().clone();
            stack.setAmount(4);
            return stack;
        })
    };

    private static final int TOTAL_WEIGHT = totalWeight();

    private static int totalWeight() {
        int sum = 0;
        for (Entry e : POOL) {
            sum += e.weight;
        }
        return sum;
    }

    /**
     * Rolls a single reward from the weighted pool.
     *
     * @param random
     *            the source of randomness (a seeded {@link Random} gives a deterministic sequence)
     *
     * @return a {@link MysteryReward} with a fresh {@link ItemStack} and the jackpot flag
     */
    @Nonnull
    public static MysteryReward roll(@Nonnull Random random) {
        int pick = random.nextInt(TOTAL_WEIGHT);

        for (Entry e : POOL) {
            pick -= e.weight;
            if (pick < 0) {
                return new MysteryReward(e.supplier.get(), e.rarity == Rarity.JACKPOT);
            }
        }

        // Unreachable (pick < TOTAL_WEIGHT); defensive fallback.
        return new MysteryReward(POOL[POOL.length - 1].supplier.get(), false);
    }

    /**
     * @return every {@link Material} any pool entry can produce (used by tests)
     */
    @Nonnull
    public static Set<Material> poolMaterials() {
        Set<Material> materials = new HashSet<>();
        for (Entry e : POOL) {
            materials.add(e.supplier.get().getType());
        }
        return materials;
    }

    /**
     * @param material
     *            the material to test
     *
     * @return whether the material is produced by a common (vanilla) pool entry
     */
    public static boolean isCommonMaterial(@Nonnull Material material) {
        for (Entry e : POOL) {
            if (e.rarity == Rarity.COMMON && e.supplier.get().getType() == material) {
                return true;
            }
        }
        return false;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `./gradlew test --tests "io.github.thebusybiscuit.slimefun4.implementation.items.blocks.TestMysteryLoot"`
Expected: PASS (3 tests). Note: the test loads MockBukkit + the Slimefun plugin so `SlimefunItems.*.item()` resolves.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/io/github/thebusybiscuit/slimefun4/implementation/items/blocks/MysteryLoot.java \
        src/test/java/io/github/thebusybiscuit/slimefun4/implementation/items/blocks/TestMysteryLoot.java
git commit -m "feat: add MysteryLoot weighted reward roller for Mystery Box

Claude-Session: https://claude.ai/code/session_013YFwH85eQ1mJXua7Hqcu1k"
```

---

### Task 2: MysteryBox block + MYSTERY_BOX/MYSTERY_KEY items + registration

**Files:**
- Create: `src/main/java/io/github/thebusybiscuit/slimefun4/implementation/items/blocks/MysteryBox.java`
- Modify: `src/main/java/io/github/thebusybiscuit/slimefun4/implementation/SlimefunItems.java` (add `MYSTERY_BOX` + `MYSTERY_KEY` near the other blocks, e.g. after the `/* Multiblocks */` region or near misc blocks ~line 600)
- Modify: `src/main/java/io/github/thebusybiscuit/slimefun4/implementation/setup/SlimefunItemSetup.java` (import `MysteryBox`; register both items with recipes near other magical-gadget items)
- Test: `src/test/java/io/github/thebusybiscuit/slimefun4/implementation/items/blocks/TestMysteryBox.java`

**Interfaces:**
- Consumes: `MysteryLoot.roll(Random)` from Task 1.
- Produces: `public class MysteryBox extends SlimefunItem implements InventoryBlock` with constructor `MysteryBox(ItemGroup, SlimefunItemStack, RecipeType, ItemStack[])`; `getInputSlots()` → `{11}`; `getOutputSlots()` → `{15}`; package-visible `boolean isMysteryKey(ItemStack)`. `SlimefunItems.MYSTERY_BOX`, `SlimefunItems.MYSTERY_KEY`.

- [ ] **Step 1: Write the failing test**

```java
package io.github.thebusybiscuit.slimefun4.implementation.items.blocks;

import org.bukkit.Material;
import org.bukkit.inventory.ItemStack;
import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.Assertions;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import io.github.thebusybiscuit.slimefun4.api.items.SlimefunItem;
import io.github.thebusybiscuit.slimefun4.api.recipes.RecipeType;
import io.github.thebusybiscuit.slimefun4.implementation.Slimefun;
import io.github.thebusybiscuit.slimefun4.implementation.SlimefunItems;
import io.github.thebusybiscuit.slimefun4.test.TestUtilities;

import org.mockbukkit.mockbukkit.MockBukkit;
import org.mockbukkit.mockbukkit.ServerMock;

class TestMysteryBox {

    private static ServerMock server;
    private static Slimefun plugin;

    @BeforeAll
    public static void load() {
        server = MockBukkit.mock();
        plugin = MockBukkit.load(Slimefun.class);
    }

    @AfterAll
    public static void unload() {
        MockBukkit.unmock();
    }

    @Test
    @DisplayName("MysteryBox registers and exposes the specified slots")
    void testRegistration() {
        MysteryBox box = new MysteryBox(TestUtilities.getItemGroup(plugin, "test"), SlimefunItems.MYSTERY_BOX,
            RecipeType.NULL, new ItemStack[9]);
        box.register(plugin);

        Assertions.assertNotNull(SlimefunItem.getById("MYSTERY_BOX"));
        Assertions.assertArrayEquals(new int[] { 11 }, box.getInputSlots());
        Assertions.assertArrayEquals(new int[] { 15 }, box.getOutputSlots());
    }

    @Test
    @DisplayName("isMysteryKey accepts a Mystery Key and rejects a bare tripwire hook")
    void testKeyDetection() {
        MysteryBox box = (MysteryBox) SlimefunItem.getById("MYSTERY_BOX");
        Assertions.assertNotNull(box);

        Assertions.assertTrue(box.isMysteryKey(SlimefunItems.MYSTERY_KEY.item()), "a Mystery Key should be accepted");
        Assertions.assertFalse(box.isMysteryKey(new ItemStack(Material.TRIPWIRE_HOOK)), "a bare tripwire hook is not a key");
        Assertions.assertFalse(box.isMysteryKey(null), "null is not a key");
    }
}
```

If `TestUtilities.getItemGroup` / `RecipeType.NULL` differ, mirror the existing `TestAutoFisher` construction (same repo, same pattern).

- [ ] **Step 2: Run test to verify it fails**

Run: `./gradlew test --tests "io.github.thebusybiscuit.slimefun4.implementation.items.blocks.TestMysteryBox"`
Expected: FAIL — `MysteryBox` / `SlimefunItems.MYSTERY_BOX` do not exist.

- [ ] **Step 3a: Add items to SlimefunItems.java**

Add near the misc/blocks items (e.g. just before the `/* Multiblocks */` region comment, around line 583):

```java
    /* Mystery Box */
    public static final SlimefunItemStack MYSTERY_BOX = new SlimefunItemStack("MYSTERY_BOX", Material.ENDER_CHEST, "<#C77DFF>Mystery Box", "", "<#F1FAEE>Insert a <#C77DFF>Mystery Key <#F1FAEE>and spin", "<#F1FAEE>for a random reward!", "", "<#E9C46A>✦ Jackpot chance!");
    public static final SlimefunItemStack MYSTERY_KEY = new SlimefunItemStack("MYSTERY_KEY", Material.TRIPWIRE_HOOK, "<#C77DFF>Mystery Key", "", "<#F1FAEE>Spin a <#C77DFF>Mystery Box <#F1FAEE>with this");
```

- [ ] **Step 3b: Write the MysteryBox class**

```java
package io.github.thebusybiscuit.slimefun4.implementation.items.blocks;

import java.util.concurrent.ThreadLocalRandom;

import javax.annotation.Nonnull;
import javax.annotation.ParametersAreNonnullByDefault;

import org.bukkit.Bukkit;
import org.bukkit.Color;
import org.bukkit.Material;
import org.bukkit.Sound;
import org.bukkit.block.Block;
import org.bukkit.entity.Player;
import org.bukkit.inventory.ItemStack;

import dev.yanianz.sudo.common.MiniMessages;
import dev.yanianz.sudo.items.CustomItemStack;
import dev.yanianz.sudo.protection.Interaction;
import io.github.thebusybiscuit.slimefun4.api.items.ItemGroup;
import io.github.thebusybiscuit.slimefun4.api.items.SlimefunItem;
import io.github.thebusybiscuit.slimefun4.api.items.SlimefunItemStack;
import io.github.thebusybiscuit.slimefun4.api.recipes.RecipeType;
import io.github.thebusybiscuit.slimefun4.core.handlers.BlockBreakHandler;
import io.github.thebusybiscuit.slimefun4.implementation.Slimefun;
import io.github.thebusybiscuit.slimefun4.implementation.SlimefunItems;
import io.github.thebusybiscuit.slimefun4.implementation.handlers.SimpleBlockBreakHandler;
import io.github.thebusybiscuit.slimefun4.utils.ChestMenuUtils;
import io.github.thebusybiscuit.slimefun4.utils.FireworkUtils;
import io.github.thebusybiscuit.slimefun4.utils.SlimefunUtils;

import me.mrCookieSlime.Slimefun.Objects.SlimefunItem.interfaces.InventoryBlock;
import me.mrCookieSlime.Slimefun.api.BlockStorage;
import me.mrCookieSlime.Slimefun.api.inventory.BlockMenu;
import me.mrCookieSlime.Slimefun.api.inventory.BlockMenuPreset;
import me.mrCookieSlime.Slimefun.api.item_transport.ItemTransportFlow;

/**
 * The {@link MysteryBox} is a non-electric magical block. The player inserts a Mystery Key and
 * clicks the Spin button; the block consumes the key, rolls a weighted reward via {@link MysteryLoot}
 * and pushes it into the output slot with a firework + sound. Jackpots broadcast to the server.
 *
 * <p>The Spin handler is attached per block in {@link BlockMenuPreset#newInstance} so it can capture
 * the block, and runs on the block's region thread (Folia-safe). The firework is spawned through the
 * region-scheduled {@link FireworkUtils}.
 *
 * @author Yan
 *
 * @see MysteryLoot
 */
public class MysteryBox extends SlimefunItem implements InventoryBlock {

    private static final int[] BORDER = { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 12, 14, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26 };
    private static final int INPUT_SLOT = 11;
    private static final int SPIN_SLOT = 13;
    private static final int OUTPUT_SLOT = 15;

    private static final ItemStack SPIN_BUTTON = CustomItemStack.create(Material.NETHER_STAR, "<#C77DFF>✦ Spin", "", "<#F1FAEE>Insert a <#C77DFF>Mystery Key <#F1FAEE>above,", "<#F1FAEE>then click to spin!");

    @ParametersAreNonnullByDefault
    public MysteryBox(ItemGroup itemGroup, SlimefunItemStack item, RecipeType recipeType, ItemStack[] recipe) {
        super(itemGroup, item, recipeType, recipe);

        addItemHandler(onBreak());

        new BlockMenuPreset(getId(), "<#C77DFF>Mystery Box") {

            @Override
            public void init() {
                constructMenu(this);
            }

            @Override
            public boolean canOpen(Block b, Player p) {
                return p.hasPermission("slimefun.inventory.bypass")
                    || Slimefun.instance().isUnitTest()
                    || Slimefun.getProtectionManager().hasPermission(p, b.getLocation(), Interaction.INTERACT_BLOCK);
            }

            @Override
            public void newInstance(BlockMenu menu, Block b) {
                menu.addMenuClickHandler(SPIN_SLOT, (p, slot, item, action) -> {
                    spin(p, b, menu);
                    return false;
                });
            }

            @Override
            public int[] getSlotsAccessedByItemTransport(ItemTransportFlow flow) {
                return flow == ItemTransportFlow.INSERT ? getInputSlots() : getOutputSlots();
            }
        };
    }

    private void constructMenu(BlockMenuPreset preset) {
        for (int i : BORDER) {
            preset.addItem(i, ChestMenuUtils.getBackground(), ChestMenuUtils.getEmptyClickHandler());
        }

        preset.addItem(INPUT_SLOT, ChestMenuUtils.getInputSlotTexture(), ChestMenuUtils.getEmptyClickHandler());
        preset.addItem(SPIN_SLOT, SPIN_BUTTON, ChestMenuUtils.getEmptyClickHandler());
        preset.addItem(OUTPUT_SLOT, ChestMenuUtils.getOutputSlotTexture(), ChestMenuUtils.getEmptyClickHandler());
    }

    @Nonnull
    private BlockBreakHandler onBreak() {
        return new SimpleBlockBreakHandler() {

            @Override
            public void onBlockBreak(Block b) {
                BlockMenu inv = BlockStorage.getInventory(b);

                if (inv != null) {
                    inv.dropItems(b.getLocation(), getInputSlots());
                    inv.dropItems(b.getLocation(), getOutputSlots());
                }
            }
        };
    }

    @ParametersAreNonnullByDefault
    private void spin(Player p, Block b, BlockMenu menu) {
        ItemStack key = menu.getItemInSlot(INPUT_SLOT);

        if (!isMysteryKey(key)) {
            Slimefun.getLocalization().sendMessage(p, "messages.wrong-item", true);
            return;
        }

        MysteryLoot.MysteryReward reward = MysteryLoot.roll(ThreadLocalRandom.current());
        ItemStack leftover = menu.pushItem(reward.item(), getOutputSlots());

        if (leftover != null) {
            // Output is full — keep the key, drop no reward.
            Slimefun.getLocalization().sendMessage(p, "messages.full-inventory", true);
            return;
        }

        menu.consumeItem(INPUT_SLOT, 1);

        FireworkUtils.launchFirework(b.getLocation().add(0.5D, 1.0D, 0.5D), Color.PURPLE);
        b.getWorld().playSound(b.getLocation(), Sound.BLOCK_NOTE_BLOCK_PLING, 1.0F, 1.4F);

        if (reward.jackpot()) {
            b.getWorld().playSound(b.getLocation(), Sound.UI_TOAST_CHALLENGE_COMPLETE, 1.0F, 1.0F);
            FireworkUtils.launchFirework(b.getLocation().add(0.5D, 1.0D, 0.5D), Color.YELLOW);
            Bukkit.broadcast(MiniMessages.component("<#E9C46A>✦ " + p.getName() + " <#F1FAEE>hit the <#C77DFF>Mystery Box <#E9C46A>jackpot!"));
        }
    }

    boolean isMysteryKey(ItemStack item) {
        return item != null && SlimefunUtils.isItemSimilar(item, SlimefunItems.MYSTERY_KEY, true);
    }

    @Override
    public int[] getInputSlots() {
        return new int[] { INPUT_SLOT };
    }

    @Override
    public int[] getOutputSlots() {
        return new int[] { OUTPUT_SLOT };
    }
}
```

Notes for the implementer:
- `messages.wrong-item` and `messages.full-inventory` are existing keys in `languages/en/messages.yml` (verified) reused as generic feedback — no new language keys.
- `SlimefunUtils.isItemSimilar(ItemStack, ItemStack, boolean)` accepts a `SlimefunItemStack` as the second arg (it is an `ItemStack`).
- `Sound` constant names (`BLOCK_NOTE_BLOCK_PLING`, `UI_TOAST_CHALLENGE_COMPLETE`) must exist on the Paper API in use; if a name differs, pick the nearest existing constant.

- [ ] **Step 3c: Register in SlimefunItemSetup.java**

Add the import with the other `...items.blocks` imports:

```java
import io.github.thebusybiscuit.slimefun4.implementation.items.blocks.MysteryBox;
```

Add the registration near other `itemGroups.magicalGadgets` items:

```java
        new MysteryBox(itemGroups.magicalGadgets, SlimefunItems.MYSTERY_BOX, RecipeType.ENHANCED_CRAFTING_TABLE,
                new ItemStack[] {
                    SlimefunItems.GOLD_24K.item(), new ItemStack(Material.DIAMOND), SlimefunItems.GOLD_24K.item(),
                    new ItemStack(Material.DIAMOND), new ItemStack(Material.ENDER_CHEST), new ItemStack(Material.DIAMOND),
                    SlimefunItems.GOLD_24K.item(), new ItemStack(Material.DIAMOND), SlimefunItems.GOLD_24K.item()
                }).register(plugin);

        new SlimefunItem(itemGroups.magicalGadgets, SlimefunItems.MYSTERY_KEY, RecipeType.ENHANCED_CRAFTING_TABLE,
                new ItemStack[] {
                    null, new ItemStack(Material.GOLD_INGOT), null,
                    new ItemStack(Material.GOLD_INGOT), new ItemStack(Material.TRIPWIRE_HOOK), new ItemStack(Material.GOLD_INGOT),
                    null, new ItemStack(Material.GOLD_INGOT), null
                }, new SlimefunItemStack(SlimefunItems.MYSTERY_KEY, 2)).register(plugin);
```

Ensure `SlimefunItem` and `SlimefunItemStack` are already imported in `SlimefunItemSetup.java` (they are — used throughout).

- [ ] **Step 4: Run tests to verify they pass**

Run: `./gradlew test --tests "io.github.thebusybiscuit.slimefun4.implementation.items.blocks.TestMysteryBox" --tests "io.github.thebusybiscuit.slimefun4.implementation.items.blocks.TestMysteryLoot"`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/io/github/thebusybiscuit/slimefun4/implementation/items/blocks/MysteryBox.java \
        src/main/java/io/github/thebusybiscuit/slimefun4/implementation/SlimefunItems.java \
        src/main/java/io/github/thebusybiscuit/slimefun4/implementation/setup/SlimefunItemSetup.java \
        src/test/java/io/github/thebusybiscuit/slimefun4/implementation/items/blocks/TestMysteryBox.java
git commit -m "feat: add Mystery Box block + Mystery Key (spin for weighted reward)

Claude-Session: https://claude.ai/code/session_013YFwH85eQ1mJXua7Hqcu1k"
```

---

### Task 3: Research gate

**Files:**
- Modify: `src/main/java/io/github/thebusybiscuit/slimefun4/implementation/setup/ResearchSetup.java`
- Modify: `src/main/resources/languages/en/researches.yml`

**Interfaces:**
- Consumes: `SlimefunItems.MYSTERY_BOX`, `SlimefunItems.MYSTERY_KEY` from Task 2.
- Produces: research keyed `"mystery_box"`, id `283`, cost `25`.

- [ ] **Step 1: Add the research registration**

Add after the `auto_fisher` registration (~line 207):

```java
        register("mystery_box", 283, "A Box of Mysteries", 25, SlimefunItems.MYSTERY_BOX, SlimefunItems.MYSTERY_KEY);
```

- [ ] **Step 2: Add the translation**

In `src/main/resources/languages/en/researches.yml`, after the `auto_fisher` line:

```yaml
  mystery_box: A Box of Mysteries
```

- [ ] **Step 3: Run the full suite**

Run: `./gradlew test`
Expected: PASS — all green, including the "all researches have a translation" and research-id-uniqueness tests. Id `283` is the next free id (current max `282`); verify no other `register(` line uses `283`.

- [ ] **Step 4: Commit**

```bash
git add src/main/java/io/github/thebusybiscuit/slimefun4/implementation/setup/ResearchSetup.java \
        src/main/resources/languages/en/researches.yml
git commit -m "feat: gate Mystery Box behind a research (id 283)

Claude-Session: https://claude.ai/code/session_013YFwH85eQ1mJXua7Hqcu1k"
```

---

## Self-Review

**Spec coverage:**
- Weighted reward roller (common/rare/jackpot), deterministic, fresh instances → Task 1 (`MysteryLoot`). ✅
- Placed non-electric GUI block, key input, Spin button, output, firework + jackpot broadcast, Folia-safe handler, break-drop → Task 2 (`MysteryBox`). ✅
- `MYSTERY_BOX` + `MYSTERY_KEY` items + recipes → Task 2. ✅
- Research gate + translation → Task 3. ✅
- Tests (roll pool/rarity/fresh; registration/slots/key detection) → Tasks 1 & 2. ✅

**Placeholder scan:** The two `sendMessage` keys are flagged for the implementer to verify against existing message keys (no new key invented). Everything else is complete code.

**Type consistency:** `roll(Random)` → `MysteryReward(ItemStack, boolean)`; `isMysteryKey(ItemStack)` (package-visible); `getInputSlots(){11}`, `getOutputSlots(){15}`; BORDER(24) + INPUT(1) + SPIN(1) + OUTPUT(1) = 27. Pool weights sum to 100. ✅
