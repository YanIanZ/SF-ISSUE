# SF4 — Guide Favorites + Craftable View Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add per-player guide favorites (shift-right-click toggle, favorites tab) and a "Craftable" tab (unlocked items with a have/missing-materials badge) to `SurvivalSlimefunGuide`.

**Architecture:** Favorites persist in the existing per-player `Config` (`data-storage/Slimefun/Players/<uuid>.yml`) on `PlayerProfile`. Pure filter/craftability logic lives in a testable helper `GuideCraftability`. Two new paginated guide views (`openFavorites`, `openCraftable`) reuse the existing item-render + `createHeader` + prev/next pattern (modeled on `openMainMenu`). Two header buttons (free slots 3, 5) open them. Favorited items get a ★ + hint appended in one shared render helper.

**Tech Stack:** Java 25, Paper API 26.2, SuDo `26.2.0` (Config, MiniMessages, CustomItemStack), MockBukkit tests.

## Global Constraints

- Colors are `<#hex>` MiniMessage only (no legacy `&`/`§`, no `org.bukkit.ChatColor`); item name/lore via `SlimefunItemStack.render(...)` or `CustomItemStack.create(...)` (post-SF2 system).
- Item ids are `SlimefunItem#getId()` (uppercase, stable).
- No new SuDo library changes; use the existing `Config` API (`getStringList`/`setValue`/`save`/`contains`).
- Commits end with the `Claude-Session:` trailer.
- Build + all existing 1791 tests must stay green after every task.

---

## File Structure

- Modify `api/player/PlayerProfile.java` — favorites storage (config-backed).
- Create `implementation/guide/GuideCraftability.java` — testable static helpers `isUnlocked`, `hasMaterials`.
- Modify `implementation/guide/SurvivalSlimefunGuide.java` — toggle, ★/hint decoration, `openFavorites`, `openCraftable`, header buttons.
- Modify `src/main/resources/languages/en/messages.yml` — new keys.
- Create tests: `TestPlayerProfileFavorites.java`, `TestGuideCraftability.java`.

---

### Task 1: PlayerProfile favorites storage

**Files:**
- Modify: `src/main/java/io/github/thebusybiscuit/slimefun4/api/player/PlayerProfile.java`
- Test: `src/test/java/io/github/thebusybiscuit/slimefun4/api/player/TestPlayerProfileFavorites.java`

**Interfaces:**
- Produces: `Set<String> getFavorites()`, `boolean isFavorite(String id)`, `void setFavorite(String id, boolean favorite)`.

- [ ] **Step 1: Write the failing test**

```java
// TestPlayerProfileFavorites.java
package io.github.thebusybiscuit.slimefun4.api.player;

import org.bukkit.entity.Player;
import org.junit.jupiter.api.*;
import org.mockbukkit.mockbukkit.MockBukkit;
import io.github.thebusybiscuit.slimefun4.implementation.Slimefun;
import io.github.thebusybiscuit.slimefun4.test.TestUtilities;
import static org.junit.jupiter.api.Assertions.*;

class TestPlayerProfileFavorites {
    private static Slimefun plugin;
    @BeforeAll static void load() { MockBukkit.mock(); plugin = MockBukkit.load(Slimefun.class); }
    @AfterAll static void unload() { MockBukkit.unmock(); }

    @Test void testFavoriteToggleAndQuery() throws InterruptedException {
        Player p = TestUtilities.randomPlayer(MockBukkit.getMock());
        PlayerProfile profile = TestUtilities.awaitProfile(p);
        assertFalse(profile.isFavorite("TEST_ITEM"));
        profile.setFavorite("TEST_ITEM", true);
        assertTrue(profile.isFavorite("TEST_ITEM"));
        assertTrue(profile.getFavorites().contains("TEST_ITEM"));
        profile.setFavorite("TEST_ITEM", false);
        assertFalse(profile.isFavorite("TEST_ITEM"));
    }
}
```
(If `TestUtilities.awaitProfile` doesn't exist, use `PlayerProfile.get(p, ...)` with a `CountDownLatch`, mirroring an existing profile test — grep `PlayerProfile.get(` in src/test for the pattern.)

- [ ] **Step 2: Run it, verify it fails** — `./gradlew test --tests "*TestPlayerProfileFavorites*"` → FAIL (methods missing).

- [ ] **Step 3: Implement in PlayerProfile.java**

Add a lazily-loaded set + methods (place near the research methods):

```java
private Set<String> favorites;

private @Nonnull Set<String> favorites() {
    if (favorites == null) {
        favorites = ConcurrentHashMap.newKeySet();
        if (configFile.contains("favorites")) {
            favorites.addAll(configFile.getStringList("favorites"));
        }
    }
    return favorites;
}

public @Nonnull Set<String> getFavorites() {
    return Set.copyOf(favorites());
}

public boolean isFavorite(@Nonnull String id) {
    return favorites().contains(id);
}

public void setFavorite(@Nonnull String id, boolean favorite) {
    Set<String> set = favorites();
    boolean changed = favorite ? set.add(id) : set.remove(id);
    if (changed) {
        configFile.setValue("favorites", new ArrayList<>(set));
        configFile.save();
    }
}
```
Add imports as needed (`java.util.Set`, `ArrayList`, `ConcurrentHashMap` — likely already imported).

- [ ] **Step 4: Run the test, verify it passes** — `./gradlew test --tests "*TestPlayerProfileFavorites*"` → PASS.

- [ ] **Step 5: Commit** — `git add` the two files; `git commit` "feat(guide): SF4 T1 — PlayerProfile favorites storage".

---

### Task 2: Craftability helper (`isUnlocked`, `hasMaterials`)

**Files:**
- Create: `src/main/java/io/github/thebusybiscuit/slimefun4/implementation/guide/GuideCraftability.java`
- Test: `src/test/java/io/github/thebusybiscuit/slimefun4/implementation/guide/TestGuideCraftability.java`

**Interfaces:**
- Produces: `static boolean isUnlocked(PlayerProfile profile, SlimefunItem item)`, `static boolean hasMaterials(Player p, SlimefunItem item)`.

- [ ] **Step 1: Write the failing test**

```java
// TestGuideCraftability.java — uses MockBukkit + TestUtilities.mockSlimefunItem
// isUnlocked: item with null research -> true; hasMaterials: inventory with the exact
// recipe ingredient -> true, empty inventory -> false.
```
(Model on existing item tests: `TestUtilities.mockSlimefunItem(plugin, "ID", CustomItemStack.create(Material.X, "<#hex>Name"))`, set a recipe via the item's constructor/`setRecipe`. Grep `mockSlimefunItem` + `setRecipe` in src/test for the exact helper signatures.)

- [ ] **Step 2: Run it, verify it fails.**

- [ ] **Step 3: Implement GuideCraftability.java**

```java
public final class GuideCraftability {
    private GuideCraftability() {}

    public static boolean isUnlocked(@Nonnull PlayerProfile profile, @Nonnull SlimefunItem item) {
        Research research = item.getResearch();
        return research == null || profile.hasUnlocked(research);
    }

    public static boolean hasMaterials(@Nonnull Player p, @Nonnull SlimefunItem item) {
        ItemStack[] recipe = item.getRecipe();
        if (recipe == null) {
            return false;
        }
        Inventory inv = p.getInventory();
        for (ItemStack ingredient : recipe) {
            if (ingredient == null) {
                continue;
            }
            if (!SlimefunUtils.containsAtLeast(inv, ingredient, ingredient.getAmount())) {
                return false;
            }
        }
        return true;
    }
}
```
If `SlimefunUtils.containsAtLeast` (Slimefun-item-aware contains) doesn't exist, add a small private helper that scans `inv.getContents()` and counts matches via `SlimefunUtils.isItemSimilar(stack, ingredient, true, false)` — grep `isItemSimilar` for the signature. Do NOT use vanilla `Inventory#containsAtLeast` alone (it ignores Slimefun item identity).

- [ ] **Step 4: Run the test, verify it passes.**

- [ ] **Step 5: Commit** — "feat(guide): SF4 T2 — craftability + unlock helpers".

---

### Task 3: Favorite toggle + ★/hint decoration

**Files:**
- Modify: `implementation/guide/SurvivalSlimefunGuide.java`
- Modify: `src/main/resources/languages/en/messages.yml`

**Interfaces:**
- Consumes: `PlayerProfile.isFavorite/setFavorite` (T1).
- Produces: a `decorateGuideLore(List<String> lore, SlimefunItem item, PlayerProfile profile)` render helper used by all item views.

- [ ] **Step 1: Add localization keys** to `messages.yml` under `guide:`

```yaml
  favorites:
    added: "<#2A9D8F>Added to favorites."
    removed: "<#8D99AE>Removed from favorites."
    star: "<#F4A261>★ Favorite"
    hint: "<#8D99AE>⇨ Shift-Right-Click to favorite"
```

- [ ] **Step 2: Add the decoration helper** (private, in SurvivalSlimefunGuide)

```java
private void appendFavoriteLore(List<String> lore, SlimefunItem item, PlayerProfile profile) {
    if (profile.isFavorite(item.getId())) {
        lore.add(SlimefunItemStack.render(Slimefun.getLocalization().getMessage(profile.getPlayer(), "guide.favorites.star")));
    }
    lore.add(SlimefunItemStack.render(Slimefun.getLocalization().getMessage(profile.getPlayer(), "guide.favorites.hint")));
}
```

- [ ] **Step 3: Wire the shift-right-click toggle** into the survival item click handler (the block near `displayItem(profile, sfitem, true)` in `showItemGroup`/the group render at ~line 305). At the top of the handler:

```java
if (action.isShiftClicked() && action.isRightClicked()) {
    boolean now = !profile.isFavorite(sfitem.getId());
    profile.setFavorite(sfitem.getId(), now);
    SoundEffect.GUIDE_BUTTON_CLICK_SOUND.playFor(pl);
    Slimefun.getLocalization().sendMessage(pl, now ? "guide.favorites.added" : "guide.favorites.removed");
    return false;
}
```
(Apply the same toggle in the search/favorites/craftable item handlers built in later tasks — keep it a small private method `toggleFavorite(Player pl, PlayerProfile profile, SlimefunItem item)` to avoid duplication.)

- [ ] **Step 4: Build + existing tests** — `./gradlew build` green (UI toggle is runtime-verified; no new unit test — T1 covers persistence).

- [ ] **Step 5: Commit** — "feat(guide): SF4 T3 — favorite toggle + lore decoration".

---

### Task 4: Favorites view + header button

**Files:**
- Modify: `implementation/guide/SurvivalSlimefunGuide.java`
- Modify: `messages.yml`
- Test: `src/test/java/.../guide/TestGuideFavoritesView.java` (smoke)

**Interfaces:**
- Produces: `void openFavorites(PlayerProfile profile, int page)`.

- [ ] **Step 1: Add keys** — `guide.title.favorites`, `guide.favorites.empty`, favorites button name/lore.

- [ ] **Step 2: Write the smoke test** — open favorites for a profile with 0 and with 2 favorites; assert no throw and the menu builds. (Model on an existing guide-open test; grep `openMainMenu`/`TestGuideOpening` in src/test.)

- [ ] **Step 3: Implement `openFavorites(profile, page)`** — paginated view modeled on `openMainMenu`:
  - `createHeader(...)`, back button.
  - Resolve `profile.getFavorites()` → `SlimefunItem.getById(id)` (skip null/disabled/hidden/inaccessible).
  - Page the resolved list (36/page); for each, `CustomItemStack.create(item.getItem(), meta -> { lore = new ArrayList<>(); appendFavoriteLore(lore, item, profile); meta.setLore(render each); })`; click → survival `displayItem`, shift-right-click → `toggleFavorite` + refresh (`openFavorites(profile, page)`).
  - Prev/next at slots 46/52 (reuse `ChestMenuUtils.getPreviousButton/getNextButton`).
  - Empty state: centered item with `guide.favorites.empty`.

- [ ] **Step 4: Add the header button** in `createHeader` slot 3:

```java
menu.addItem(3, CustomItemStack.create(Material.NETHER_STAR, "<#F4A261>" + Slimefun.getLocalization().getMessage(p, "guide.favorites.button")));
menu.addMenuClickHandler(3, (pl, slot, item, action) -> { openFavorites(profile, 1); return false; });
```

- [ ] **Step 5: Run tests + build** → green.

- [ ] **Step 6: Commit** — "feat(guide): SF4 T4 — favorites view + header button".

---

### Task 5: Craftable view + header button

**Files:**
- Modify: `implementation/guide/SurvivalSlimefunGuide.java`
- Modify: `messages.yml`
- Test: `src/test/java/.../guide/TestGuideCraftableView.java` (smoke)

**Interfaces:**
- Consumes: `GuideCraftability.isUnlocked/hasMaterials` (T2), `appendFavoriteLore` (T3).
- Produces: `void openCraftable(PlayerProfile profile, int page)`.

- [ ] **Step 1: Add keys** — `guide.title.craftable`, `guide.craftable.has-materials` (`<#2A9D8F>✔ You have the materials`), `guide.craftable.missing-materials` (`<#E76F51>✘ Missing materials`), craftable button name/lore.

- [ ] **Step 2: Write the smoke test** — build the craftable view for a profile; assert no throw; with a mock item whose recipe the player holds, `GuideCraftability.hasMaterials` is true (already covered in T2 — here just assert the view opens).

- [ ] **Step 3: Implement `openCraftable(profile, page)`**:
  - Filter `getEnabledSlimefunItems()` → `!hidden && isItemGroupAccessible(p, item) && GuideCraftability.isUnlocked(profile, item)`.
  - Page (36/page). For each shown item: `lore` = badge line (`hasMaterials(p, item)` ? has-materials : missing-materials) + `appendFavoriteLore`. Render via `CustomItemStack.create(item.getItem(), meta -> setLore(...))`.
  - Click → `displayItem`; shift-right-click → `toggleFavorite` + refresh.
  - Prev/next 46/52; `createHeader`; back button; empty state.
  - **Perf:** compute `hasMaterials` only for the ≤36 items on the shown page (inside the page loop), not for the full unlocked list.

- [ ] **Step 4: Header button** in `createHeader` slot 5:

```java
menu.addItem(5, CustomItemStack.create(Material.CRAFTING_TABLE, "<#2A9D8F>" + Slimefun.getLocalization().getMessage(p, "guide.craftable.button")));
menu.addMenuClickHandler(5, (pl, slot, item, action) -> { openCraftable(profile, 1); return false; });
```

- [ ] **Step 5: Run all tests + build** → green (1791 + new).

- [ ] **Step 6: Commit** — "feat(guide): SF4 T5 — craftable view + materials badge".

---

## Self-Review

- **Spec coverage:** favorites storage (T1), unlock/materials logic (T2), toggle + ★/hint (T3), favorites tab + button (T4), craftable tab + badge + button (T5) — all spec sections covered.
- **Type consistency:** `openFavorites`/`openCraftable(PlayerProfile, int)`, `GuideCraftability.isUnlocked(PlayerProfile, SlimefunItem)` / `hasMaterials(Player, SlimefunItem)`, `PlayerProfile.isFavorite/setFavorite/getFavorites` — consistent across tasks.
- **Perf:** materials badge is page-scoped (≤36 items, on open). Favorites lazily loaded once.
- **Verify-at-impl (grep first):** `TestUtilities.awaitProfile`/profile-get pattern; `mockSlimefunItem`/`setRecipe` signatures; `SlimefunUtils.containsAtLeast`/`isItemSimilar` signatures; `Slimefun.getLocalization().getMessage(Player, key)` signature; `ChestMenuUtils.getPreviousButton/getNextButton` signatures. Adjust the sketched code to the real signatures.
