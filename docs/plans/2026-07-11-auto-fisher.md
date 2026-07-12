# Auto-Fisher Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an END_GAME electric Auto-Fisher machine that auto-fishes a curated weighted loot table while touching water, with an optional bait boost.

**Architecture:** `AutoFisher extends SlimefunItem implements InventoryBlock, EnergyNetComponent` mirroring `AbstractGrowthAccelerator` (chest menu + energy buffer + synchronized `BlockTicker` + break handler). Loot is a pure `FishingLoot` helper, unit-tested with a seeded `Random`. The machine is registered in `SlimefunItemSetup` with a recipe and gated by a research in `ResearchSetup`.

**Tech Stack:** Java 17, Slimefun4 API, `dev.yanianz.sudo` (SuDo) `CustomItemStack`, Paper API, MockBukkit + JUnit 5 for tests, Gradle.

## Global Constraints

- **Colors:** 100% hex MiniMessage — every name/lore string uses `<#RRGGBB>` tags. No `org.bukkit.ChatColor`, no legacy `§`/`&` codes in source.
- **Item stacks:** build display items with `dev.yanianz.sudo.items.CustomItemStack` — `CustomItemStack.create(Material, name, lore...)` (bridged sink, MiniMessage-renders name + lore). Menu decoration items go through it too.
- **Folia-safe:** the `BlockTicker` MUST return `isSynchronized() == true`. All world reads and inventory writes run on the block's region thread. No async world access.
- **SuDo + Paper API only.** No new third-party dependencies.
- **Energy:** implement `EnergyNetComponent` as `CONSUMER`; use `getCharge(Location)` / `removeCharge(Location, int)`.
- **Build/test:** `./gradlew test` (from repo root `/Users/rheninxy/Client-Project/Slimefun4`). Trust Gradle, not the LSP. `.gradle/` is untracked — never stage it.
- **Commits:** end every commit message with `Claude-Session: https://claude.ai/code/session_013YFwH85eQ1mJXua7Hqcu1k`. Stage only the explicit files each task names — never `git add -A`.

---

### Task 1: FishingLoot — pure weighted loot roller

**Files:**
- Create: `src/main/java/io/github/thebusybiscuit/slimefun4/implementation/items/electric/machines/FishingLoot.java`
- Test: `src/test/java/io/github/thebusybiscuit/slimefun4/implementation/items/electric/machines/TestFishingLoot.java`

**Interfaces:**
- Consumes: nothing (pure).
- Produces: `public static ItemStack FishingLoot.roll(Random random, boolean baited)` — returns a fresh (cloned) `ItemStack` chosen proportional to weight from the unbaited or baited pool. Also `public static boolean FishingLoot.isTreasure(Material)` for tests + parity (returns true for the 5 treasure materials).

- [ ] **Step 1: Write the failing test**

```java
package io.github.thebusybiscuit.slimefun4.implementation.items.electric.machines;

import java.util.Random;

import org.bukkit.Material;
import org.bukkit.inventory.ItemStack;
import org.junit.jupiter.api.Assertions;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

class TestFishingLoot {

    private static final java.util.Set<Material> POOL = java.util.Set.of(
        Material.COD, Material.SALMON, Material.TROPICAL_FISH, Material.PUFFERFISH,
        Material.ENCHANTED_BOOK, Material.NAUTILUS_SHELL, Material.NAME_TAG, Material.SADDLE, Material.BOW,
        Material.LEATHER_BOOTS, Material.STICK, Material.BOWL, Material.ROTTEN_FLESH, Material.LILY_PAD);

    @Test
    @DisplayName("roll returns a non-null ItemStack from the declared pool")
    void testRollInPool() {
        Random r = new Random(1234L);
        for (int i = 0; i < 10_000; i++) {
            ItemStack result = FishingLoot.roll(r, false);
            Assertions.assertNotNull(result);
            Assertions.assertTrue(POOL.contains(result.getType()), "unexpected material: " + result.getType());
        }
    }

    @Test
    @DisplayName("roll returns a fresh instance each call (mutation-safe)")
    void testRollFreshInstance() {
        Random r = new Random(42L);
        ItemStack a = FishingLoot.roll(r, false);
        a.setAmount(64);
        ItemStack b = FishingLoot.roll(new Random(42L), false);
        Assertions.assertEquals(1, b.getAmount(), "roll must return a fresh clone, not a shared prototype");
    }

    @Test
    @DisplayName("baited pool yields more treasure than unbaited over N rolls")
    void testBaitedMoreTreasure() {
        int baitedTreasure = 0;
        int plainTreasure = 0;
        Random rb = new Random(7L);
        Random rp = new Random(7L);
        for (int i = 0; i < 20_000; i++) {
            if (FishingLoot.isTreasure(FishingLoot.roll(rb, true).getType())) baitedTreasure++;
            if (FishingLoot.isTreasure(FishingLoot.roll(rp, false).getType())) plainTreasure++;
        }
        Assertions.assertTrue(baitedTreasure > plainTreasure,
            "baited treasure (" + baitedTreasure + ") should exceed unbaited (" + plainTreasure + ")");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `./gradlew test --tests "io.github.thebusybiscuit.slimefun4.implementation.items.electric.machines.TestFishingLoot"`
Expected: FAIL — `FishingLoot` does not exist / cannot find symbol.

- [ ] **Step 3: Write minimal implementation**

```java
package io.github.thebusybiscuit.slimefun4.implementation.items.electric.machines;

import java.util.Random;

import javax.annotation.Nonnull;
import javax.annotation.ParametersAreNonnullByDefault;

import org.bukkit.Material;
import org.bukkit.inventory.ItemStack;

/**
 * A pure, deterministic weighted loot roller for the {@link AutoFisher}.
 *
 * <p>Two static weighted pools mirror vanilla fishing proportions: an unbaited
 * pool (mostly fish) and a baited pool that shifts odds toward treasure. Rolls
 * are deterministic for a given {@link Random} sequence, which keeps the loot
 * unit-testable without a live server.
 */
public final class FishingLoot {

    private FishingLoot() {}

    // Treasure materials — shared by both pools; also used by isTreasure().
    private static final Material[] TREASURE = {
        Material.ENCHANTED_BOOK, Material.NAUTILUS_SHELL, Material.NAME_TAG, Material.SADDLE, Material.BOW
    };

    private static final class Entry {
        private final Material material;
        private final int weight;

        private Entry(Material material, int weight) {
            this.material = material;
            this.weight = weight;
        }
    }

    // Unbaited: Common 85 (30+25+17+13), Treasure 5 (1x5), Junk 10 (2x5) -> total 100.
    private static final Entry[] UNBAITED = {
        new Entry(Material.COD, 30), new Entry(Material.SALMON, 25),
        new Entry(Material.TROPICAL_FISH, 17), new Entry(Material.PUFFERFISH, 13),
        new Entry(Material.ENCHANTED_BOOK, 1), new Entry(Material.NAUTILUS_SHELL, 1),
        new Entry(Material.NAME_TAG, 1), new Entry(Material.SADDLE, 1), new Entry(Material.BOW, 1),
        new Entry(Material.LEATHER_BOOTS, 2), new Entry(Material.STICK, 2), new Entry(Material.BOWL, 2),
        new Entry(Material.ROTTEN_FLESH, 2), new Entry(Material.LILY_PAD, 2)
    };

    // Baited: Common 84 (30+24+17+13), Treasure 12 (~2-3 each), Junk 4 (~1 each) -> total 100.
    private static final Entry[] BAITED = {
        new Entry(Material.COD, 30), new Entry(Material.SALMON, 24),
        new Entry(Material.TROPICAL_FISH, 17), new Entry(Material.PUFFERFISH, 13),
        new Entry(Material.ENCHANTED_BOOK, 3), new Entry(Material.NAUTILUS_SHELL, 3),
        new Entry(Material.NAME_TAG, 2), new Entry(Material.SADDLE, 2), new Entry(Material.BOW, 2),
        new Entry(Material.LEATHER_BOOTS, 1), new Entry(Material.STICK, 1), new Entry(Material.BOWL, 1),
        new Entry(Material.ROTTEN_FLESH, 1)
    };

    private static final int UNBAITED_TOTAL = totalWeight(UNBAITED);
    private static final int BAITED_TOTAL = totalWeight(BAITED);

    private static int totalWeight(Entry[] pool) {
        int sum = 0;
        for (Entry e : pool) {
            sum += e.weight;
        }
        return sum;
    }

    @Nonnull
    @ParametersAreNonnullByDefault
    public static ItemStack roll(Random random, boolean baited) {
        Entry[] pool = baited ? BAITED : UNBAITED;
        int total = baited ? BAITED_TOTAL : UNBAITED_TOTAL;
        int pick = random.nextInt(total);

        for (Entry e : pool) {
            pick -= e.weight;
            if (pick < 0) {
                return new ItemStack(e.material);
            }
        }

        // Unreachable (pick < total), but keep the compiler + defensive path happy.
        return new ItemStack(pool[pool.length - 1].material);
    }

    public static boolean isTreasure(@Nonnull Material material) {
        for (Material m : TREASURE) {
            if (m == material) {
                return true;
            }
        }
        return false;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `./gradlew test --tests "io.github.thebusybiscuit.slimefun4.implementation.items.electric.machines.TestFishingLoot"`
Expected: PASS (3 tests).

- [ ] **Step 5: Commit**

```bash
git add src/main/java/io/github/thebusybiscuit/slimefun4/implementation/items/electric/machines/FishingLoot.java \
        src/test/java/io/github/thebusybiscuit/slimefun4/implementation/items/electric/machines/TestFishingLoot.java
git commit -m "feat: add FishingLoot weighted loot roller for Auto-Fisher

Claude-Session: https://claude.ai/code/session_013YFwH85eQ1mJXua7Hqcu1k"
```

---

### Task 2: AUTO_FISHER item + AutoFisher machine class

**Files:**
- Create: `src/main/java/io/github/thebusybiscuit/slimefun4/implementation/items/electric/machines/AutoFisher.java`
- Modify: `src/main/java/io/github/thebusybiscuit/slimefun4/implementation/SlimefunItems.java` (add `AUTO_FISHER` field near the other machines, e.g. just after `AUTO_BREEDER` ~line 791)
- Modify: `src/main/java/io/github/thebusybiscuit/slimefun4/implementation/setup/SlimefunItemSetup.java` (import `AutoFisher`; register near the accelerators ~line 2375)
- Test: `src/test/java/io/github/thebusybiscuit/slimefun4/implementation/items/electric/machines/TestAutoFisher.java`

**Interfaces:**
- Consumes: `FishingLoot.roll(Random, boolean)` from Task 1.
- Produces: `public class AutoFisher extends SlimefunItem implements InventoryBlock, EnergyNetComponent` with a constructor `AutoFisher(ItemGroup, SlimefunItemStack, RecipeType, ItemStack[])`; `getInputSlots()` → `{10,11}`; `getOutputSlots()` → `{14,15,16,23,24,25}`; `getCapacity()` → `512`. `SlimefunItems.AUTO_FISHER` (a `SlimefunItemStack`, id `"AUTO_FISHER"`).

- [ ] **Step 1: Write the failing test**

```java
package io.github.thebusybiscuit.slimefun4.implementation.items.electric.machines;

import org.bukkit.Location;
import org.bukkit.Material;
import org.bukkit.World;
import org.bukkit.block.Block;
import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.Assertions;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import io.github.thebusybiscuit.slimefun4.api.items.SlimefunItem;
import io.github.thebusybiscuit.slimefun4.implementation.Slimefun;
import io.github.thebusybiscuit.slimefun4.implementation.SlimefunItems;
import io.github.thebusybiscuit.slimefun4.test.TestUtilities;

import org.mockbukkit.mockbukkit.MockBukkit;
import org.mockbukkit.mockbukkit.ServerMock;

class TestAutoFisher {

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
    @DisplayName("AutoFisher registers and exposes the specified slots")
    void testRegistration() {
        AutoFisher fisher = new AutoFisher(TestUtilities.getItemGroup(plugin, "test"), SlimefunItems.AUTO_FISHER,
            io.github.thebusybiscuit.slimefun4.api.recipes.RecipeType.NULL, new org.bukkit.inventory.ItemStack[9]);
        fisher.register(plugin);

        Assertions.assertNotNull(SlimefunItem.getById("AUTO_FISHER"));
        Assertions.assertArrayEquals(new int[] {10, 11}, fisher.getInputSlots());
        Assertions.assertArrayEquals(new int[] {14, 15, 16, 23, 24, 25}, fisher.getOutputSlots());
        Assertions.assertEquals(512, fisher.getCapacity());
    }

    @Test
    @DisplayName("hasWater is true with water below, false when dry")
    void testHasWater() {
        AutoFisher fisher = (AutoFisher) SlimefunItem.getById("AUTO_FISHER");
        Assertions.assertNotNull(fisher);

        World world = server.addSimpleWorld("fisher-world");
        Block dry = world.getBlockAt(0, 64, 0);
        Block wet = world.getBlockAt(10, 64, 0);
        wet.getRelative(org.bukkit.block.BlockFace.DOWN).setType(Material.WATER);

        Assertions.assertTrue(fisher.hasWater(wet), "block with water below should report water");
        Assertions.assertFalse(fisher.hasWater(dry), "isolated block should report no water");
    }
}
```

Note: if `TestUtilities.getItemGroup(...)` does not exist, use the same item-group construction the neighbouring machine tests use (check `src/test/java/.../TestUtilities.java` and an existing machine test); the point of the test is registration + slot arrays + water check, not the group specifics. `hasWater` must be package-visible (not private) for the test — declare it `boolean hasWater(Block)`.

- [ ] **Step 2: Run test to verify it fails**

Run: `./gradlew test --tests "io.github.thebusybiscuit.slimefun4.implementation.items.electric.machines.TestAutoFisher"`
Expected: FAIL — `AutoFisher` / `SlimefunItems.AUTO_FISHER` do not exist.

- [ ] **Step 3a: Add the AUTO_FISHER item to SlimefunItems.java**

Insert after the `AUTO_BREEDER` field (~line 791):

```java
    public static final SlimefunItemStack AUTO_FISHER = new SlimefunItemStack("AUTO_FISHER", Material.PRISMARINE, "<#48CAE4>Auto-Fisher", "", "<#F1FAEE>Auto-fishes while touching water", "", LoreBuilder.machine(MachineTier.END_GAME, MachineType.MACHINE), LoreBuilder.powerBuffer(512), "<#4A4E69>⇨ <#8D99AE>Bait (raw cod/salmon) doubles speed & luck");
```

- [ ] **Step 3b: Write the AutoFisher machine class**

```java
package io.github.thebusybiscuit.slimefun4.implementation.items.electric.machines;

import java.util.concurrent.ThreadLocalRandom;

import javax.annotation.Nonnull;
import javax.annotation.ParametersAreNonnullByDefault;

import org.bukkit.Material;
import org.bukkit.block.Block;
import org.bukkit.block.BlockFace;
import org.bukkit.block.data.BlockData;
import org.bukkit.block.data.Waterlogged;
import org.bukkit.inventory.ItemStack;

import dev.yanianz.sudo.items.CustomItemStack;
import io.github.thebusybiscuit.slimefun4.api.items.ItemGroup;
import io.github.thebusybiscuit.slimefun4.api.items.SlimefunItem;
import io.github.thebusybiscuit.slimefun4.api.items.SlimefunItemStack;
import io.github.thebusybiscuit.slimefun4.api.recipes.RecipeType;
import io.github.thebusybiscuit.slimefun4.core.attributes.EnergyNetComponent;
import io.github.thebusybiscuit.slimefun4.core.handlers.BlockBreakHandler;
import io.github.thebusybiscuit.slimefun4.core.networks.energy.EnergyNetComponentType;
import io.github.thebusybiscuit.slimefun4.implementation.handlers.SimpleBlockBreakHandler;
import io.github.thebusybiscuit.slimefun4.utils.ChestMenuUtils;

import me.mrCookieSlime.CSCoreLibPlugin.Configuration.Config;
import me.mrCookieSlime.Slimefun.Objects.SlimefunItem.interfaces.InventoryBlock;
import me.mrCookieSlime.Slimefun.Objects.handlers.BlockTicker;
import me.mrCookieSlime.Slimefun.api.BlockStorage;
import me.mrCookieSlime.Slimefun.api.inventory.BlockMenu;
import me.mrCookieSlime.Slimefun.api.inventory.BlockMenuPreset;

/**
 * The {@link AutoFisher} is an END_GAME electric machine that automatically
 * fishes while it is touching water, rolling a curated weighted loot table
 * (see {@link FishingLoot}) and pushing catches into its output slots.
 *
 * <p>An optional bait (raw cod or salmon) placed in an input slot halves the
 * cast interval and shifts the odds toward treasure, consuming one bait per
 * catch. The machine uses a synchronized {@link BlockTicker}, keeping all world
 * and inventory access on the block's region thread (Folia-safe).
 */
public class AutoFisher extends SlimefunItem implements InventoryBlock, EnergyNetComponent {

    private static final int[] BORDER = { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 12, 13, 17, 18, 19, 20, 21, 22, 26 };
    private static final int[] INPUT_SLOTS = { 10, 11 };
    private static final int[] OUTPUT_SLOTS = { 14, 15, 16, 23, 24, 25 };

    private static final int ENERGY_PER_CATCH = 30;
    private static final int INTERVAL_UNBAITED = 6;
    private static final int INTERVAL_BAITED = 3;
    private static final String PROGRESS_KEY = "fishing_progress";

    @ParametersAreNonnullByDefault
    public AutoFisher(ItemGroup itemGroup, SlimefunItemStack item, RecipeType recipeType, ItemStack[] recipe) {
        super(itemGroup, item, recipeType, recipe);

        addItemHandler(onBreak());
        createPreset(this, this::constructMenu);
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

    private void constructMenu(BlockMenuPreset preset) {
        for (int i : BORDER) {
            preset.addItem(i, CustomItemStack.create(Material.CYAN_STAINED_GLASS_PANE, " "), ChestMenuUtils.getEmptyClickHandler());
        }
    }

    @Override
    public EnergyNetComponentType getEnergyComponentType() {
        return EnergyNetComponentType.CONSUMER;
    }

    @Override
    public int getCapacity() {
        return 512;
    }

    @Override
    public int[] getInputSlots() {
        return INPUT_SLOTS;
    }

    @Override
    public int[] getOutputSlots() {
        return OUTPUT_SLOTS;
    }

    @Override
    public void preRegister() {
        super.preRegister();
        addItemHandler(new BlockTicker() {

            @Override
            public void tick(Block b, SlimefunItem sf, Config data) {
                AutoFisher.this.tick(b);
            }

            @Override
            public boolean isSynchronized() {
                return true;
            }
        });
    }

    private void tick(Block b) {
        if (getCharge(b.getLocation()) < ENERGY_PER_CATCH) {
            return;
        }

        if (!hasWater(b)) {
            return;
        }

        BlockMenu inv = BlockStorage.getInventory(b);
        if (inv == null) {
            return;
        }

        int baitSlot = firstBaitSlot(inv);
        boolean baited = baitSlot != -1;
        int needed = baited ? INTERVAL_BAITED : INTERVAL_UNBAITED;

        int progress = readProgress(b);

        if (progress + 1 < needed) {
            BlockStorage.addBlockInfo(b, PROGRESS_KEY, String.valueOf(progress + 1));
            return;
        }

        ItemStack loot = FishingLoot.roll(ThreadLocalRandom.current(), baited);
        ItemStack leftover = inv.pushItem(loot, getOutputSlots());

        if (leftover != null) {
            // Output is full: hold state — no energy drain, no bait consumed, no reset.
            return;
        }

        removeCharge(b.getLocation(), ENERGY_PER_CATCH);

        if (baited) {
            inv.consumeItem(baitSlot);
        }

        BlockStorage.addBlockInfo(b, PROGRESS_KEY, "0");
    }

    private int readProgress(Block b) {
        String raw = BlockStorage.getLocationInfo(b.getLocation(), PROGRESS_KEY);

        if (raw == null) {
            return 0;
        }

        try {
            return Integer.parseInt(raw);
        } catch (NumberFormatException e) {
            return 0;
        }
    }

    private int firstBaitSlot(BlockMenu inv) {
        for (int slot : getInputSlots()) {
            ItemStack item = inv.getItemInSlot(slot);

            if (isBait(item)) {
                return slot;
            }
        }

        return -1;
    }

    private boolean isBait(ItemStack item) {
        if (item == null) {
            return false;
        }

        if (item.getType() != Material.COD && item.getType() != Material.SALMON) {
            return false;
        }

        // Do not eat a Slimefun item that happens to be shaped like a cod.
        return SlimefunItem.getByItem(item) == null;
    }

    boolean hasWater(Block b) {
        if (isWater(b.getRelative(BlockFace.DOWN))) {
            return true;
        }

        for (BlockFace face : new BlockFace[] { BlockFace.NORTH, BlockFace.EAST, BlockFace.SOUTH, BlockFace.WEST }) {
            if (isWater(b.getRelative(face))) {
                return true;
            }
        }

        return false;
    }

    private boolean isWater(Block b) {
        if (b.getType() == Material.WATER) {
            return true;
        }

        BlockData data = b.getBlockData();
        return data instanceof Waterlogged && ((Waterlogged) data).isWaterlogged();
    }
}
```

- [ ] **Step 3c: Register the machine in SlimefunItemSetup.java**

Add the import (with the other `...machines` imports near line 114):

```java
import io.github.thebusybiscuit.slimefun4.implementation.items.electric.machines.AutoFisher;
```

Add the registration near the accelerators (~line 2375, after the `AnimalGrowthAccelerator` block):

```java
        new AutoFisher(itemGroups.electricity, SlimefunItems.AUTO_FISHER, RecipeType.ENHANCED_CRAFTING_TABLE,
                new ItemStack[] {
                    new ItemStack(Material.FISHING_ROD), new ItemStack(Material.PRISMARINE_SHARD), new ItemStack(Material.FISHING_ROD),
                    SlimefunItems.ELECTRIC_MOTOR.item(), new ItemStack(Material.WATER_BUCKET), SlimefunItems.ELECTRIC_MOTOR.item(),
                    SlimefunItems.REINFORCED_ALLOY_INGOT.item(), SlimefunItems.BIG_CAPACITOR.item(), SlimefunItems.REINFORCED_ALLOY_INGOT.item()
                }).register(plugin);
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `./gradlew test --tests "io.github.thebusybiscuit.slimefun4.implementation.items.electric.machines.TestAutoFisher" --tests "io.github.thebusybiscuit.slimefun4.implementation.items.electric.machines.TestFishingLoot"`
Expected: PASS. If `TestUtilities.getItemGroup` doesn't exist, adapt the test's item-group construction to match an existing machine test (this is a test-only adjustment; the machine code above is unaffected).

- [ ] **Step 5: Commit**

```bash
git add src/main/java/io/github/thebusybiscuit/slimefun4/implementation/items/electric/machines/AutoFisher.java \
        src/main/java/io/github/thebusybiscuit/slimefun4/implementation/SlimefunItems.java \
        src/main/java/io/github/thebusybiscuit/slimefun4/implementation/setup/SlimefunItemSetup.java \
        src/test/java/io/github/thebusybiscuit/slimefun4/implementation/items/electric/machines/TestAutoFisher.java
git commit -m "feat: add Auto-Fisher machine (water-check, bait boost, Folia-safe tick)

Claude-Session: https://claude.ai/code/session_013YFwH85eQ1mJXua7Hqcu1k"
```

---

### Task 3: Research gate

**Files:**
- Modify: `src/main/java/io/github/thebusybiscuit/slimefun4/implementation/setup/ResearchSetup.java` (add one `register(...)` line near the other machine researches, e.g. after the `auto_breeder` line ~206)

**Interfaces:**
- Consumes: `SlimefunItems.AUTO_FISHER` from Task 2.
- Produces: a registered research keyed `"auto_fisher"`, id `282`, cost `25`.

- [ ] **Step 1: Add the research registration**

Add after the `auto_breeder` registration (~line 206):

```java
        register("auto_fisher", 282, "The Art of Automated Angling", 25, SlimefunItems.AUTO_FISHER);
```

- [ ] **Step 2: Run the research + full test suite**

Run: `./gradlew test`
Expected: PASS — all tests green, including any research-count / research-id-uniqueness tests. If a test asserts the exact number of researches or a specific id set, update that fixture to include id `282` (research `auto_fisher`). Id `282` is the next free id (current max is `281`); verify no other `register(` line already uses `282`.

- [ ] **Step 3: Commit**

```bash
git add src/main/java/io/github/thebusybiscuit/slimefun4/implementation/setup/ResearchSetup.java
git commit -m "feat: gate Auto-Fisher behind a research (id 282)

Claude-Session: https://claude.ai/code/session_013YFwH85eQ1mJXua7Hqcu1k"
```

---

## Self-Review

**Spec coverage:**
- Curated weighted loot (fish/treasure/junk), baited vs unbaited → Task 1 (`FishingLoot`). ✅
- Machine class, menu slots, energy buffer, water check, bait, Folia-safe sync ticker, break-drop → Task 2 (`AutoFisher`). ✅
- `AUTO_FISHER` item + recipe registration → Task 2. ✅
- Research gate → Task 3. ✅
- Tests (loot determinism, baited>unbaited treasure, fresh instance; registration, slots, water check) → Tasks 1 & 2. ✅

**Placeholder scan:** All code blocks are complete. The two flagged adaptation points (`TestUtilities.getItemGroup` fallback, research-count fixture update) are conditional test-only adjustments with explicit instructions, not placeholders in shipped code.

**Type consistency:** `roll(Random, boolean)`, `isTreasure(Material)`, `hasWater(Block)` (package-visible), `getInputSlots(){10,11}`, `getOutputSlots(){14,15,16,23,24,25}`, `getCapacity()=512`, `ENERGY_PER_CATCH=30` — consistent across tasks and matching the spec. Slot math: BORDER(19) + INPUT(2) + OUTPUT(6) = 27. ✅
