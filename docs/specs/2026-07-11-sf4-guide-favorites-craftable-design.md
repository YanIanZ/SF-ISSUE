# Slimefun4 SF4 — Guide Favorites + "Craftable" View

**Date:** 2026-07-11
**Status:** Draft — awaiting user review
**Repo:** `/Users/rheninxy/Client-Project/Slimefun4`
**Target:** Java 25, Paper API 26.2, SuDo `26.2.0`, Folia-aware
**Phase:** SF4 — first new-gameplay slice (after SF1–SF3 migration arc)

## Context

The migration arc (SF1 Paper/SuDo bump, SF2 color/dough, SF3 Paper API) is complete. SF4 begins
"real new mechanics." Direction chosen: **Progression / QoL**; concrete mechanic: **Guide
favorites + a "craftable" view**. Both are player-facing additions to the existing
`SurvivalSlimefunGuide`, backed by the existing per-player `PlayerProfile` config.

The guide already has the exact reusable machinery:
- `SurvivalSlimefunGuide.openSearch(...)` iterates `Slimefun.getRegistry().getEnabledSlimefunItems()`,
  filters, and renders a paginated `ChestMenu` — the template for the two new views.
- `createHeader(...)` builds the 0–8 header row (slot 1 = menu/settings, slot 7 = search; **slots
  3 and 5 are free**).
- Guide item click handlers receive a `ClickAction` with `isShiftClicked()` / `isRightClicked()`.
- `PlayerProfile` persists per-player data in a SuDo `Config`
  (`data-storage/Slimefun/Players/<uuid>.yml`) with `markDirty()` / `save()` — the same mechanism
  research uses, and where favorites will live.
- `SlimefunItem.getRecipe()` returns the `ItemStack[9]` recipe → the basis for the materials badge.

## Goals

1. Players can **favorite** any item from the guide (shift + right-click) and view favorites in a
   dedicated tab.
2. A **"Craftable" view** lists every item the player has **unlocked** (research unlocked, or no
   research), each annotated with a **materials badge** (do they currently have the ingredients).
3. Favorites persist per-player across sessions.
4. `./gradlew build` green; new logic covered by tests; no perf regression on guide open.

## Non-Goals

- No new craftable **items** — this is guide UX only.
- No auto-crafting from the view (it's informational).
- No cross-player favorite sharing.
- The materials badge is **not** a live inventory watcher — it's computed when the view opens (per
  shown page), not continuously.

## Design

### 1. Favorites storage — `api/player/PlayerProfile.java`

Add a config-backed favorite set, mirroring how research is stored.

- Field: `private final Set<String> favorites = ConcurrentHashMap.newKeySet();`
- On profile load (where research is loaded from `configFile`): read the string list at key
  `"favorites"` into the set.
- Methods:
  - `public @Nonnull Set<String> getFavorites()` — unmodifiable copy.
  - `public boolean isFavorite(@Nonnull String id)`.
  - `public void setFavorite(@Nonnull String id, boolean favorite)` — add/remove, write the list
    back to `configFile` at `"favorites"`, `markDirty()`.

Keys are Slimefun item ids (`SlimefunItem#getId()`), uppercase — stable across restarts.

### 2. Favorite toggle — `SurvivalSlimefunGuide` item click handler

In the survival item click handler (the block around `displayItem(profile, sfitem, true)`), before
the normal open:

```java
if (action.isShiftClicked() && action.isRightClicked()) {
    boolean nowFavorite = !profile.isFavorite(sfitem.getId());
    profile.setFavorite(sfitem.getId(), nowFavorite);
    SoundEffect.GUIDE_BUTTON_CLICK_SOUND.playFor(pl);
    Slimefun.getLocalization().sendMessage(pl, nowFavorite ? "guide.favorites.added" : "guide.favorites.removed");
    // re-render current page so the star indicator updates
    openMainMenu(profile, ...currentPage...); // or refresh the current view
    return false;
}
```

Discoverability + feedback:
- Favorited items get a **★ indicator** appended to their guide-display lore (a single localized
  line, e.g. `<#F4A261>★ Favorite`) wherever items are rendered in guide views (a small helper
  `decorateGuideItem(ItemStack, SlimefunItem, PlayerProfile)` applied in the render path).
- A lore hint line `<#8D99AE>⇨ Shift-Right-Click to favorite` on guide items (same helper), so the
  gesture is discoverable without a tutorial.

### 3. Favorites view — `openFavorites(PlayerProfile profile, int page)`

Modeled on `openSearch`:
- Resolve each id in `profile.getFavorites()` via `SlimefunItem.getById(id)`, skipping
  null/disabled/hidden or group-inaccessible items.
- Render a paginated `ChestMenu` (reuse the search render + prev/next buttons + `createHeader`).
- Each item click → `displayItem(profile, sfitem, true)`; shift-right-click → toggle (un-favorite,
  which removes it from this view on refresh).
- Empty state: a centered "No favorites yet — shift-right-click items to add them" item.

### 4. "Craftable" view — `openCraftable(PlayerProfile profile, int page)`

- Iterate `getEnabledSlimefunItems()`, keep items that are **unlocked**: `isUnlocked(p, item)` =
  `item.getResearch() == null || profile.hasUnlocked(item.getResearch())` **and** the item is
  group-accessible + not hidden.
- For each item **shown on the current page**, compute the **materials badge**:
  `hasMaterials(Player, SlimefunItem)` = every non-null ingredient of `item.getRecipe()` is present
  in the player's inventory (`inventory.containsAtLeast(ingredient, ingredient.getAmount())`, honoring
  Slimefun item identity via `SlimefunUtils.isItemSimilar`). Append a localized lore badge:
  `<#2A9D8F>✔ You have the materials` or `<#E76F51>✘ Missing materials`.
- Paginated `ChestMenu` + `createHeader`. Compute the badge only for the ≤36 items on the shown page
  (not all unlocked items) to bound cost.

### 5. Header buttons — `createHeader(...)`

Add two buttons to the header row (free slots 3 and 5):
- Slot 3 — **Favorites** button (e.g. `Material.NETHER_STAR`, name `<#F4A261>Favorites`) →
  `openFavorites(profile, 1)`.
- Slot 5 — **Craftable** button (e.g. `Material.CRAFTING_TABLE`, name `<#2A9D8F>Craftable`) →
  `openCraftable(profile, 1)`.

Buttons built via `CustomItemStack.create(...)` (SuDo) with `<#hex>` names/lore (consistent with the
post-SF2 color system).

### 6. Localization — `src/main/resources/languages/en/messages.yml` (+ guide file)

New keys (English; other locales fall back): `guide.favorites.title`, `guide.favorites.added`,
`guide.favorites.removed`, `guide.favorites.empty`, `guide.favorites.hint`, `guide.favorites.star`,
`guide.craftable.title`, `guide.craftable.has-materials`, `guide.craftable.missing-materials`,
plus the two button names/lores.

## Testing

- **PlayerProfile favorites unit test** (MockBukkit): set/isFavorite/getFavorites round-trips;
  persistence — write favorites, reload the `Config`, assert they survive; removing works.
- **`hasMaterials` unit test**: a mock player inventory with/without ingredients → correct badge;
  respects Slimefun item identity (a vanilla look-alike is not accepted).
- **`isUnlocked` filter test**: item with no research → unlocked; item with unlocked research →
  unlocked; item with locked research → excluded.
- **Guide open smoke test**: `openFavorites`/`openCraftable` build a menu without throwing for empty
  and non-empty cases (MockBukkit player + profile).
- `./gradlew build` green; existing 1791 tests still pass.
- **RUNTIME-QA caveat:** the actual in-guide UX (shift-right-click toggle, star indicator, badge
  rendering, pagination) needs a live server.

## Success Criteria

1. Shift-right-click toggles favorite (persisted); favorited items show a ★ in guide views.
2. Favorites tab lists favorited items; Craftable tab lists unlocked items with an accurate
   have/missing-materials badge.
3. Two header buttons open the views; empty states handled.
4. Build + all tests green; badge cost bounded to the shown page.

## Risks

- **Materials-badge cost:** `hasMaterials` scans the inventory per shown item. Bounded to ≤36
  items/page and computed only on open (not per tick), so it's a one-off O(page × recipe) scan.
  Mitigation: page-scoped computation; if still heavy, cache per (player, item) for the menu's
  lifetime.
- **Favorites referencing removed/disabled items** (addon uninstalled): resolve defensively
  (`getById` null-check, skip), and prune such ids opportunistically on load.
- **Guide render coupling:** the ★/hint decoration touches the shared item-render path — keep it in
  one helper so all three views (groups, search, favorites, craftable) stay consistent and the
  change is localized.
- **Persistence format:** favorites is a plain string list under `"favorites"` in the existing
  per-player yml — additive, no migration of existing profiles needed (absent key = empty set).
