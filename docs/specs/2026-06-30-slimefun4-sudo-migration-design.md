# Slimefun4 Dough + Star → SuDo Migration

**Date:** 2026-06-30
**Approach:** Big Bang — replace all imports in one step

## Goal

Replace two dependencies (`dough-api` + `star-api`) with a single dependency (`sudo-api`), using SuDo APIs throughout.

## Scope

| From | To | Files |
|---|---|---|
| `com.github.Slimefun.dough:dough-api:cb22e71335` | `dev.yanianz:sudo-api:26.1.2` | 33 |
| `com.github.YanIanZ.Star:star-api:v2.0.2` | (removed — replaced by SuDo) | 201 |

**232 Java files** affected. **29 unique imports** — **27 direct package renames**, **2 API adaptation gaps**.

## build.gradle.kts Changes

### Dependencies

Remove:
```kotlin
implementation("com.github.Slimefun.dough:dough-api:cb22e71335")
implementation("com.github.YanIanZ.Star:star-api:v2.0.2")
```

Add:
```kotlin
implementation("dev.yanianz:sudo-api:26.1.2")
```

### shadowJar Relocation

Remove:
```kotlin
relocate("io.github.bakedlibs.dough", "io.github.thebusybiscuit.slimefun4.libraries.dough")
relocate("dev.yanianz.star", "io.github.thebusybiscuit.slimefun4.libraries.star")
```

Add:
```kotlin
relocate("dev.yanianz.sudo", "io.github.thebusybiscuit.slimefun4.libraries.sudo")
```

### shadowJar Excludes

Remove all 5 dough excludes (no longer needed):
```kotlin
// REMOVE all of these:
exclude("io/github/bakedlibs/dough/items/CustomItemStack.class")
exclude("io/github/bakedlibs/dough/skins/CustomGameProfile.class")
exclude("io/github/bakedlibs/dough/skins/PlayerHead.class")
exclude("io/github/bakedlibs/dough/config/Config.class")
exclude("io/github/bakedlibs/dough/common/ChatColors.class")
```

## Import Renames (27 direct)

### Dough → SuDo (2 imports, 33 files)

```
io.github.bakedlibs.dough.protection.Interaction       → dev.yanianz.sudo.protection.Interaction
io.github.bakedlibs.dough.protection.ProtectionManager  → dev.yanianz.sudo.protection.ProtectionManager
```

### Star → SuDo (25 imports, 201 files)

```
dev.yanianz.star.blocks.BlockPosition           → dev.yanianz.sudo.blocks.BlockPosition
dev.yanianz.star.blocks.ChunkPosition           → dev.yanianz.sudo.blocks.ChunkPosition
dev.yanianz.star.blocks.Vein                    → dev.yanianz.sudo.blocks.Vein
dev.yanianz.star.chat.ChatInput                 → dev.yanianz.sudo.chat.ChatInput
dev.yanianz.star.collections.KeyMap             → dev.yanianz.sudo.collections.KeyMap
dev.yanianz.star.collections.LoopIterator       → dev.yanianz.sudo.collections.LoopIterator
dev.yanianz.star.collections.OptionalMap        → dev.yanianz.sudo.collections.OptionalMap
dev.yanianz.star.collections.RandomizedSet      → dev.yanianz.sudo.collections.RandomizedSet
dev.yanianz.star.common.CommonPatterns          → dev.yanianz.sudo.common.CommonPatterns
dev.yanianz.star.common.PlayerList              → dev.yanianz.sudo.common.PlayerList
dev.yanianz.star.config.Config                  → dev.yanianz.sudo.config.Config
dev.yanianz.star.data.TriStateOptional          → dev.yanianz.sudo.data.TriStateOptional
dev.yanianz.star.data.persistent.PersistentDataAPI → dev.yanianz.sudo.data.persistent.PersistentDataAPI
dev.yanianz.star.inventory.InvUtils             → dev.yanianz.sudo.inventory.InvUtils
dev.yanianz.star.items.CustomItemStack          → dev.yanianz.sudo.items.CustomItemStack
dev.yanianz.star.items.ItemMetaSnapshot         → dev.yanianz.sudo.items.ItemMetaSnapshot
dev.yanianz.star.items.ItemStackEditor          → dev.yanianz.sudo.items.ItemStackEditor
dev.yanianz.star.items.ItemUtils                → dev.yanianz.sudo.items.ItemUtils
dev.yanianz.star.recipes.MinecraftRecipe        → dev.yanianz.sudo.recipes.MinecraftRecipe
dev.yanianz.star.recipes.RecipeSnapshot         → dev.yanianz.sudo.recipes.RecipeSnapshot
dev.yanianz.star.scheduling.TaskQueue           → dev.yanianz.sudo.scheduling.TaskQueue
dev.yanianz.star.skins.PlayerHead               → dev.yanianz.sudo.skins.PlayerHead
dev.yanianz.star.skins.PlayerSkin               → dev.yanianz.sudo.skins.PlayerSkin
dev.yanianz.star.skins.UUIDLookup               → dev.yanianz.sudo.skins.UUIDLookup
dev.yanianz.star.skins.nms.PlayerHeadAdapter    → dev.yanianz.sudo.skins.nms.PlayerHeadAdapter
dev.yanianz.star.updater.BlobBuildUpdater       → dev.yanianz.sudo.updater.BlobBuildUpdater
dev.yanianz.star.updater.PluginUpdater           → dev.yanianz.sudo.updater.PluginUpdater
dev.yanianz.star.versions.PrefixedVersion       → dev.yanianz.sudo.versions.PrefixedVersion
```

## Gap 1: ChatColors → MiniMessages

**Problem:** `dev.yanianz.star.common.ChatColors` has no SuDo equivalent. ~50 call sites use:
- `ChatColors.color("&aText")` — translates `&` color codes to legacy `§` strings
- `ChatColors.alternating(text, ChatColor...)` — alternating Bukkit ChatColors per character
- `star.common.ChatColors` only does `ChatColor.translateAlternateColorCodes('&', input)`

**Solution:** Convert to Adventure Component via MiniMessage format.
SuDo's `MiniMessages.component("...")` accepts MiniMessage tags (`<green>`, `<red>`, `<bold>`, etc).

Color code mapping:
| Legacy `&` | MiniMessage | Legacy `&` | MiniMessage |
|---|---|---|---|
| `&a` | `<green>` | `&c` | `<red>` |
| `&e` | `<yellow>` | `&b` | `<aqua>` |
| `&6` | `<gold>` | `&7` | `<gray>` |
| `&l` | `<bold>` | `&o` | `<italic>` |
| `&n` | `<underline>` | `&r` | `<reset>` |

Examples:
| Before | After |
|---|---|
| `ChatColors.color("&aText")` | `MiniMessages.component("<green>Text")` |
| `ChatColors.color("&c&lError")` | `MiniMessages.component("<red><bold>Error")` |
| `ChatColors.color("&7info")` | `MiniMessages.component("<gray>info")` |

For `ChatColors.alternating(text, ChatColor...)` → use `MiniMessages.alternating(text, NamedTextColor...)`.

The `libraries/dough/common/ChatColors.java` wrapper file will be **deleted**. All call sites rewritten directly.

## Gap 2: PlayerHead API Adaptation

**Problem:** Slimefun4's vendored `PlayerHead.java` uses reflection to call:
- `skin.getProfile()` → then `apply.invoke(skin.getProfile(), meta)` (reflection, star-style)
- `customProfile.getGameProfile()` → reflection to extract GameProfile for `setSkin()`

SuDo's `PlayerSkin` uses `toProfile("CS-CoreLib")` which returns `CustomGameProfile` (extends GameProfile)
with a public `apply(SkullMeta)` method. No reflection needed.

**Fix:** Replace reflection with direct API:
- `skin.getProfile()...apply(meta)` → `skin.toProfile("CS-CoreLib").apply(meta)`
- `customProfile.getGameProfile()` → `skin.toProfile("CS-CoreLib")` (already extends GameProfile)

## Vendored Wrapper File Changes

5 local files under `src/main/java/io/github/thebusybiscuit/slimefun4/libraries/dough/`:

| File | Action |
|---|---|
| `config/Config.java` | Update import: `dev.yanianz.star.config.Config` → `dev.yanianz.sudo.config.Config` |
| `common/ChatColors.java` | **Delete** — all callers converted to `MiniMessages` |
| `items/CustomItemStack.java` | Update import: `dev.yanianz.star.items.CustomItemStack` → `dev.yanianz.sudo.items.CustomItemStack` |
| `skins/PlayerHead.java` | Update imports + adapt reflection API to `toProfile().apply(meta)` |
| `skins/CustomGameProfile.java` | Update import: `dev.yanianz.star.skins.nms.PlayerHeadAdapter` → `dev.yanianz.sudo.skins.nms.PlayerHeadAdapter` |

## Execution Order

1. Update `build.gradle.kts` (deps, relocation, excludes)
2. Replace all `import io.github.bakedlibs.dough.protection.*` → `import dev.yanianz.sudo.protection.*`
3. Replace all `import dev.yanianz.star.*` → `import dev.yanianz.sudo.*`
4. Convert ~50 ChatColors call sites to MiniMessages
5. Update 4 vendored wrapper classes
6. Adapt `PlayerHead.java` reflection → `toProfile()`
7. Delete `ChatColors.java` wrapper
8. Build & test (`./gradlew build`)

## Verification

- `./gradlew build` compiles cleanly
- `./gradlew test` passes all tests
- No remaining references to `dev.yanianz.star` or `io.github.bakedlibs.dough` in source
