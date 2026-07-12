# SF4b — "Recently Viewed" Guide Tab + Favorites/Craftable Polish

> **For agentic workers:** REQUIRED SUB-SKILL: superpowers:subagent-driven-development. Steps use `- [ ]` checkboxes.

**Goal:** Add an auto-tracked "Recently Viewed" guide tab and polish the SF4a favorites/craftable features (count badge, materials-first sort, null-guard, deferred tests).

**Architecture:** Recently-viewed is a bounded ordered list persisted in the per-player `Config` (same mechanism as favorites). Views are recorded in `SurvivalSlimefunGuide.displayItem`. The tab reuses the existing paginated-view + header-button pattern (like `openFavorites`/`openCraftable`).

**Tech Stack:** Java 25, Paper 26.2, SuDo 26.2.0 (Config, MiniMessages, CustomItemStack), MockBukkit.

## Global Constraints

- `<#hex>` MiniMessage only; item ids via `getId()`; item name/lore via `SlimefunItemStack.render(...)`/`CustomItemStack.create(...)`.
- Reuse existing helpers (`toggleFavorite`, `appendFavoriteLore`, `getDecoratedItem`, `GuideCraftability`); do not duplicate.
- No SuDo library change.
- Build + all existing tests green after every task. Commits end with the `Claude-Session:` trailer.

---

## File Structure

- Modify `api/player/PlayerProfile.java` — recently-viewed storage.
- Modify `implementation/guide/SurvivalSlimefunGuide.java` — record views, `openRecent`, header button, favorites-count, craftable sort, null-guard.
- Modify `src/main/resources/languages/en/messages.yml` — new keys.
- Tests: `TestPlayerProfileRecent.java`; extend `TestPlayerProfileFavorites` (reload) + a favorites-view >36 pagination test.

---

### Task 1: PlayerProfile recently-viewed storage

**Files:** Modify `api/player/PlayerProfile.java`; Test `TestPlayerProfileRecent.java`.

**Interfaces (Produces):** `List<String> getRecentlyViewed()` (most-recent-first), `void addRecentlyViewed(String id)`.

- Bounded, ordered, de-duplicated. Constant `RECENT_LIMIT = 18`. Config key `"recently-viewed"`.
- `addRecentlyViewed(id)`: remove any existing occurrence, insert at front, trim to `RECENT_LIMIT`, write list to `configFile` + `save()`. Lazy-load on first access (mirror the favorites `favorites()` loader).
- `getRecentlyViewed()`: unmodifiable copy in most-recent-first order.

- [ ] Step 1: Failing test — toggle order/dedup/cap: add A,B,C → getRecentlyViewed()==[C,B,A]; re-add A → [A,C,B]; add 20 items → size==18, newest first. Use the real profile-get pattern (`server.addPlayer()` + `TestUtilities.awaitProfile`).
- [ ] Step 2: Run, verify fail.
- [ ] Step 3: Implement (mirror favorites: a `LinkedList<String>`/`List<String>` field, lazy-loaded from `configFile.getStringList("recently-viewed")`, front-insert + trim + save).
- [ ] Step 4: Run, verify pass.
- [ ] Step 5: Commit `feat(guide): SF4b T1 — PlayerProfile recently-viewed storage`.

---

### Task 2: Record views + Recently-Viewed tab + header button

**Files:** Modify `SurvivalSlimefunGuide.java`, `messages.yml`; Test `TestGuideRecentView.java` (smoke).

**Interfaces:** Consumes T1; Produces `void openRecent(PlayerProfile profile, int page)`.

- **Record views:** in `displayItem(profile, item, addToHistory)` (survival path), call `profile.addRecentlyViewed(item.getId())`. Grep `displayItem` for the exact method; add the call once, guarded so it doesn't fire in cheat mode if that path differs.
- **`openRecent(profile, page)`:** model on `openFavorites` — resolve `profile.getRecentlyViewed()` via `SlimefunItem.getById` (skip null/disabled/hidden/`!isItemGroupAccessible`/`isDisabledIn(p.getWorld())`), render `getDecoratedItem` clones (★/hint), left-click→`displayItem`, shift-right-click→`toggleFavorite`+refresh `openRecent(profile, page)`, prev/next 46/52, `createHeader`, back button, empty-state (`guide.recent.empty`).
- **Header button:** add to `createHeader` at a free slot (confirm free by grep: header uses 1=menu, 3=favorites, 5=craftable, 7=search, 8=wiki — use **slot 4** if free, else slot 2/6): `CustomItemStack.create(Material.CLOCK, "<#8ECAE6>" + getMessage(p, "guide.recent.button"))` → `openRecent(profile, 1)`.
- **Localization:** `guide.title.recent`, `guide.recent.button`, `guide.recent.empty`.

- [ ] Step 1: Add localization keys.
- [ ] Step 2: Smoke test — `openRecent` for 0 and ≥1 recent items doesn't throw (add via `profile.addRecentlyViewed(id)`).
- [ ] Step 3: Implement `openRecent` + the `displayItem` recording + header button.
- [ ] Step 4: `./gradlew build` green.
- [ ] Step 5: Commit `feat(guide): SF4b T2 — recently-viewed tab + view tracking`.

---

### Task 3: Favorites/Craftable polish + deferred tests

**Files:** Modify `SurvivalSlimefunGuide.java`, `messages.yml`; Tests: extend `TestPlayerProfileFavorites` + add a favorites-view pagination test.

- **Favorites-count badge:** the favorites header button name shows the count, e.g. `Favorites (N)` via `getMessage` with a `%count%` placeholder replaced by `profile.getFavorites().size()`. Add key `guide.favorites.button` already exists — append the count in the button build (grep the slot-3 button in `createHeader`).
- **Craftable materials-first sort:** in `openCraftable`, after collecting the unlocked list, stably sort so items where `GuideCraftability.hasMaterials(p, item)` is true come first (compute once per item for the SORT — note this widens the cost from page-scoped to full-list; to keep it bounded, sort ONLY the current page's slice is not possible since sort needs the whole list — so: acceptable tradeoff = compute `hasMaterials` for the full unlocked list to sort, OR keep page-scoped and skip the sort. **Chosen: sort the full unlocked list by hasMaterials; document that this makes the craftable open O(unlocked × recipe).** If perf is a concern in review, fall back to page-scoped no-sort.)
- **`appendFavoriteLore` null-guard:** add `if (p == null) return;` after `Player p = profile.getPlayer()` (or thread a guaranteed-non-null player) so the helper is safe for reuse from any view.
- **Deferred tests:** (a) extend `TestPlayerProfileFavorites` with a persistence-across-reload assertion — `setFavorite(id,true)`, construct a fresh `Config("data-storage/Slimefun/Players/<uuid>.yml")`, assert `getStringList("favorites")` contains the id. (b) Add a test that `openFavorites` handles a >36-favorite set without throwing (register 40 mock items, favorite all, `openFavorites(profile, 2)` opens page 2).

- [ ] Step 1: Failing/added tests (reload-persistence + >36 pagination).
- [ ] Step 2: Run — reload test fails if persistence is broken (it isn't — `setFavorite` saves immediately — so it should PASS; if so, it's a regression guard, keep it).
- [ ] Step 3: Implement count badge, craftable sort, null-guard.
- [ ] Step 4: Run tests + `./gradlew build` green.
- [ ] Step 5: Commit `feat(guide): SF4b T3 — favorites count, craftable sort, polish + tests`.

---

## Self-Review

- Coverage: recently-viewed storage (T1), tab + tracking (T2), favorites/craftable polish + deferred tests (T3).
- Type consistency: `openRecent(PlayerProfile, int)`; `getRecentlyViewed()`/`addRecentlyViewed(String)`; reuses `getDecoratedItem`/`toggleFavorite`/`GuideCraftability`.
- Perf note: T3's craftable sort widens `hasMaterials` from page-scoped to full-unlocked-list — flagged for review; page-scoped no-sort is the fallback.
- Verify-at-impl (grep first): `displayItem` signature + survival gating; free header slot; `getMessage(Player,key)` + placeholder-replace helper; `ChestMenuUtils.getPreviousButton/getNextButton`; the favorites slot-3 button in `createHeader`.
