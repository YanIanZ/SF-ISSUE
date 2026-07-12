# Slimefun4 SF3 — Paper API 100% (remove `[removal]` deprecations)

**Date:** 2026-07-11
**Status:** Draft — awaiting user review
**Repo:** `/Users/rheninxy/Client-Project/Slimefun4`
**Target:** Java 25, Paper API `26.2.build.56-alpha`, SuDo `26.2.0`
**Phase:** SF3 — fourth phase after the SF2 color/dough arc (complete)

## Context

SF1 bumped to Paper 26.2; the SF2 arc removed all legacy color (`org.bukkit.ChatColor`,
`net.md_5.bungee`, dough). `.spigot()` calls and Spigot/bungee imports are already **zero**.
What remains for "Paper API 100%" is the set of Bukkit/Paper APIs the compiler flags
`[removal]` (deprecated **and marked for removal** — they will break on a future Paper). A
clean compile shows **78 warnings**; of those, **~50 are Paper/Bukkit `[removal]`** (the rest
are 3rd-party integration deprecations and cosmetic `[dep-ann]`).

Inventory (Paper/Bukkit `[removal]`, verified against the 26.2 API jar):

| Removed API | Count | Replacement (verified in jar) |
|---|---|---|
| `Effect.STEP_SOUND` (via `World.playEffect`) | 28 | `spawnParticle(Particle.BLOCK, loc, n, blockData)` + `playSound(breakSound)` |
| `Effect.SMOKE` | 6 | `spawnParticle(Particle.SMOKE, loc, n, offset)` |
| `PotionMeta.getBasePotionData`/`setBasePotionData` + `org.bukkit.potion.PotionData` | 9 | `getBasePotionType`/`setBasePotionType` + `PotionType` |
| `SlimefunItemStack` deprecated delegates: `getData`/`setData`(`MaterialData`), `getTranslationKey`, `getMaxItemUseDuration`, `getRarity`(`ItemRarity`) | ~8 | **remove** (see §3) |
| `Sound.getKey()` | 1 | `sound.key().value()` (Adventure `Key`) |
| `Biome.valueOf(String)` | 1 | `Registry.BIOME.get(NamespacedKey.minecraft(...))` |
| `GameRule.DO_LIMITED_CRAFTING` | 1 | `GameRule.getByName("doLimitedCrafting")` |

## Goals

1. Zero Paper/Bukkit `[removal]` warnings in `src/main` (excluding 3rd-party integration APIs).
2. Behavior preserved (the block-break effect keeps both particles **and** the break sound).
3. `./gradlew build` green; all existing tests pass.

## Non-Goals

- **3rd-party `[removal]`** — WorldEdit `BlockVector3.getBlockX/Y/Z`, mcMMO `getPlaceStore` (4
  warnings). We cannot change their APIs; a future WorldEdit/mcMMO bump handles them.
- **Cosmetic `[dep-ann]`** — SF's own methods flagged "deprecated item not annotated with
  `@Deprecated`". Not removal-critical; out of scope.
- **Adventure-migration of String item meta** (`setDisplayName(String)`/`setLore(List<String>)`)
  — these are `[deprecation]` not `[removal]`, and are coupled to Slimefun's string read-back.
  Separate future work, not SF3.

## Design

### SF3a — `Effect.STEP_SOUND` / `Effect.SMOKE` → `spawnParticle` (34 sites)

Add a small helper in the existing `utils/compatibility/` package (mirrors `VersionedParticle`
etc.), so the 34 call sites collapse to one-liners and the semantics live in one place.

```java
// utils/compatibility/BlockEffects.java  (new)
public final class BlockEffects {
    private BlockEffects() {}

    /** Faithful replacement for the removed {@code Effect.STEP_SOUND}: block-break particles + break sound. */
    public static void playBreakEffect(@Nonnull Location loc, @Nonnull Material material) {
        World world = loc.getWorld();
        if (world == null) {
            return;
        }
        BlockData data = material.createBlockData();
        world.spawnParticle(Particle.BLOCK, loc, 10, 0.2, 0.2, 0.2, data);
        world.playSound(loc, data.getSoundGroup().getBreakSound(), 1.0F, 1.0F);
    }

    /** Replacement for the removed {@code Effect.SMOKE}: a small smoke puff. */
    public static void playSmokeEffect(@Nonnull Location loc) {
        World world = loc.getWorld();
        if (world == null) {
            return;
        }
        world.spawnParticle(Particle.SMOKE, loc, 4, 0.1, 0.1, 0.1, 0.0);
    }
}
```

Conversions:
- `world.playEffect(loc, Effect.STEP_SOUND, material)` → `BlockEffects.playBreakEffect(loc, material)`.
- `world.playEffect(loc, Effect.STEP_SOUND, 1)` (legacy int data = stone) → `BlockEffects.playBreakEffect(loc, Material.STONE)`.
- `world.playEffect(loc, Effect.SMOKE, 4 | BlockFace)` → `BlockEffects.playSmokeEffect(loc)`.

Remove the now-unused `import org.bukkit.Effect;` per file. Where the argument was
`b.getType()` etc. that resolves to a `Material`, pass it straight through.

### SF3b — `PotionData` → `PotionType` (9 sites)

`org.bukkit.potion.PotionData` and `PotionMeta.getBasePotionData/setBasePotionData` are removed;
`PotionType` + `getBasePotionType`/`setBasePotionType` (non-deprecated) replace them 1:1.
Primary site: `AutoBrewer` (potion recipe/fermentation maps already use `PotionType`; convert the
`PotionData` construction + meta read/write). Also `SlimefunUtils:500` (`getBasePotionData` read).
Drop `import org.bukkit.potion.PotionData;`.

### SF3c — remove `SlimefunItemStack` deprecated delegates (~8 warnings)

`SlimefunItemStack` re-exposes five `@Deprecated` pass-through delegates that call
Paper methods marked for removal — a delegate to a removed method cannot survive. **Remove** them
(user-approved; API break for any addon calling them, but they are already `@Deprecated` and
would break regardless):
- `getData()` / `setData(MaterialData)` — drop `import org.bukkit.material.MaterialData;`.
- `getTranslationKey()`.
- `getMaxItemUseDuration()` (the no-arg one; keep the `getMaxItemUseDuration(LivingEntity)` overload — not removed).
- `getRarity()` — drop `import io.papermc.paper.inventory.ItemRarity;`.

Before deleting, grep `src/main` + `src/test` for internal callers of each; if any exist,
convert the caller to the non-removed equivalent or inline. (Expected: none — these are
API-surface re-exports.)

### SF3d — registry / gamerule misc (3 sites)

- `SoundEffect.java:133` `sound.getKey().getKey()` → `sound.key().value()` (Adventure `Key`, non-deprecated).
- `BiomeMapParser.java:173` `Biome.valueOf(formattedValue)` → `Registry.BIOME.get(NamespacedKey.minecraft(formattedValue.toLowerCase(Locale.ROOT)))`; handle null (invalid biome) as the old `IllegalArgumentException` path did. `Biome` is now `org.bukkit.block.Biome`.
- `AutoCrafterListener.java:74` `GameRule.DO_LIMITED_CRAFTING` → `GameRule.getByName("doLimitedCrafting")` (returns `GameRule<Boolean>`).

## Testing

- `./gradlew build` — compile + all existing tests (MockBukkit).
- Warning gate: `./gradlew clean compileJava` shows **zero** `[removal]` warnings for Paper/Bukkit
  symbols (the 4 remaining 3rd-party WorldEdit/mcMMO `[removal]` warnings are acceptable/out of scope).
- MockBukkit coverage: particle/sound calls and `PotionType` meta are exercised by existing item
  tests where present; **RUNTIME-QA caveat** stands for the visual/audible effects (particles,
  break sound, smoke) — verify on a live server.
- Confirm no test referenced a removed `SlimefunItemStack` delegate (fix or drop if so).

## Success Criteria

1. Zero Paper/Bukkit `[removal]` warnings in `src/main` (3rd-party excepted).
2. `Effect.STEP_SOUND`/`SMOKE`, `PotionData`, the 5 `SlimefunItemStack` delegates, `Sound.getKey`,
   `Biome.valueOf`, `GameRule.DO_LIMITED_CRAFTING` all gone.
3. Behavior preserved (block-break = particles + sound); build + all tests green.

## Risks

- **Effect semantics drift:** `Effect.STEP_SOUND`'s exact particle count/spread isn't publicly
  specified; the helper picks reasonable values (10 particles, 0.2 spread) + the real break sound.
  Visual is approximate, not pixel-identical. Mitigation: single helper, easy to tune; RUNTIME-QA.
- **`SlimefunItemStack` delegate removal is an API break** for addons calling those five methods.
  Accepted (user-approved) — they are already `@Deprecated` and wrap removed APIs.
- **`Registry.BIOME.get` null-handling:** `Biome.valueOf` threw on an unknown name; the registry
  `get` returns null. The parser must reproduce the same "invalid biome" handling (log/skip) so a
  bad biome-map config fails the same way.
- **`GameRule.getByName` returns a raw-ish `GameRule<T>`** — cast/param carefully so
  `getGameRuleValue` still returns `Boolean`.

---

## Post-implementation addendum (2026-07-11)

**The original inventory above was truncated** — gradle's compile cache emits `[removal]`
warnings only on the *first* clean compile, and the initial capture cut off at ~20 lines. The
full src/main surface was larger. Actual outcome (commit `ff4ac11`, build + all 1791 tests green):

**Fixed (clean, non-removal replacement):** Effect.STEP_SOUND/SMOKE (34 → BlockEffects), PotionData
(9 → PotionType; dead pre-1.20.2 branches deleted), 5 SlimefunItemStack delegates removed,
Biome.valueOf (→ Registry.BIOME), FoodLevelChangeEvent (2 → 3-arg ctor), ExpCollector (→ 5-arg ctor).

**Suppressed** (`@SuppressWarnings("removal")` + documenting comment — **Paper 26.2 offers no
non-`[removal]` replacement**): GameRule.DO_LIMITED_CRAFTING (getByName also [removal]);
Sound.getKey() (Sound.key() also [removal]; Registry.SOUNDS.getKeyOrThrow throws under MockBukkit →
93 test failures); EntityDamageByEntityEvent ctors (every public ctor incl. the DamageSource one is
[removal] — Paper is removing manual construction of this event). These are honestly documented, not
silently ignored; revisit when Paper ships stable replacements.

**Out of scope (unchanged):** 3rd-party WorldEdit/mcMMO `[removal]` (4); test-only MockBukkit API
`[removal]` (~70) — `assertSucceeded`, `ItemEntityMock`, event ctors in tests, etc.

**Result:** zero *unsuppressed* Paper/Bukkit `[removal]` in src/main.
