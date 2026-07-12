# Slimefun4 Dough + Star → SuDo Migration Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace dough-api + star-api with sudo-api in Slimefun4, removing all legacy dependencies.

**Architecture:** Big Bang — single commit replacing ~232 files. Bulk automated import renames via `sed` for 29 package renames, + manual adaptation for 2 API gaps (ChatColors, PlayerHead).

**Tech Stack:** Java 25, Gradle 9.6.1+ (via wrapper), Paper API 26.1.2, Adventure 4.26.1 (transitive via sudo-api), SuDo 26.1.2

## Global Constraints

- sudo-api:26.1.2, shaded + relocated to `io.github.thebusybiscuit.slimefun4.libraries.sudo`
- Java 25, Gradle wrapper, no CI — verify build locally
- `dev.yanianz.sudo.*` imports for all SuDo APIs
- Adventure `Component` / `MiniMessages.component(...)` for user-facing text (no legacy `§` codes)
- Test classes use `Test` prefix, JUnit 5, `failOnNoDiscoveredTests = false`
- Paper API compileOnly — never upgrade casually

---

### Task 1: Update build.gradle.kts

**Files:**
- Modify: `build.gradle.kts`

**Interfaces:**
- Produces: Updated dependency declaration, relocation, and excludes for Task 2-12

- [ ] **Step 1: Replace dependencies**

Replace lines 84-85:
```kotlin
    implementation("com.github.YanIanZ.Star:star-api:v2.0.2")
    implementation("com.github.Slimefun.dough:dough-api:cb22e71335")
```
With:
```kotlin
    implementation("dev.yanianz:sudo-api:26.1.2")
```

- [ ] **Step 2: Replace relocation rules**

Replace lines 130-131:
```kotlin
    relocate("dev.yanianz.star", "io.github.thebusybiscuit.slimefun4.libraries.star")
    relocate("io.github.bakedlibs.dough", "io.github.thebusybiscuit.slimefun4.libraries.dough")
```
With:
```kotlin
    relocate("dev.yanianz.sudo", "io.github.thebusybiscuit.slimefun4.libraries.sudo")
```

- [ ] **Step 3: Remove dough class excludes**

Delete lines 136-140:
```kotlin
    exclude("io/github/bakedlibs/dough/items/CustomItemStack.class")
    exclude("io/github/bakedlibs/dough/skins/CustomGameProfile.class")
    exclude("io/github/bakedlibs/dough/skins/PlayerHead.class")
    exclude("io/github/bakedlibs/dough/config/Config.class")
    exclude("io/github/bakedlibs/dough/common/ChatColors.class")
```

- [ ] **Step 4: Commit**

```bash
git add build.gradle.kts
git commit -m "build: replace dough + star with sudo-api"
```

---

### Task 2: Bulk Rename Dough Protection Imports

**Files:**
- Modify: 33 files using `io.github.bakedlibs.dough.protection.*`

**Interfaces:**
- Consumes: Task 1 (build.gradle.kts changes)
- Produces: All dough imports replaced with sudo imports for Task 3-12

- [ ] **Step 1: Replace Interaction import**

```bash
find src -name "*.java" -exec sed -i '' 's/import io\.github\.bakedlibs\.dough\.protection\.Interaction;/import dev.yanianz.sudo.protection.Interaction;/g' {} +
```

- [ ] **Step 2: Replace ProtectionManager import**

```bash
find src -name "*.java" -exec sed -i '' 's/import io\.github\.bakedlibs\.dough\.protection\.ProtectionManager;/import dev.yanianz.sudo.protection.ProtectionManager;/g' {} +
```

- [ ] **Step 3: Replace PlayerSkin import (only in vendored wrapper)**

```bash
sed -i '' 's/import io\.github\.bakedlibs\.dough\.skins\.PlayerSkin;/import dev.yanianz.sudo.skins.PlayerSkin;/g' src/main/java/io/github/thebusybiscuit/slimefun4/libraries/dough/skins/PlayerHead.java
```

- [ ] **Step 4: Commit**

```bash
git add src/
git commit -m "refactor: replace dough imports with sudo equivalents"
```

---

### Task 3: Bulk Rename Star Imports to SuDo

**Files:**
- Modify: ~201 files using `dev.yanianz.star.*`

**Interfaces:**
- Consumes: Task 1-2
- Produces: All star imports replaced with sudo imports for Task 4-12

- [ ] **Step 1: Run bulk sed replacement**

```bash
find src -name "*.java" -exec sed -i '' 's/dev\.yanianz\.star\./dev.yanianz.sudo./g' {} +
```

- [ ] **Step 2: Commit**

```bash
git add src/
git commit -m "refactor: bulk rename star imports to sudo"
```

---

### Task 4: Update Vendored Config.java Wrapper

**Files:**
- Modify: `src/main/java/io/github/thebusybiscuit/slimefun4/libraries/dough/config/Config.java`

**Interfaces:**
- Consumes: Task 3 (imports already renamed via sed)
- Produces: Config wrapper delegates to SuDo's Config

- [ ] **Step 1: Verify import was renamed**

The sed in Task 3 already changed `dev.yanianz.star.config.Config` → `dev.yanianz.sudo.config.Config` on the `extends` clause. Verify:

```bash
grep "dev.yanianz.sudo.config.Config" src/main/java/io/github/thebusybiscuit/slimefun4/libraries/dough/config/Config.java
```
Expected: Line showing `extends dev.yanianz.sudo.config.Config`

- [ ] **Step 2: Commit**

```bash
git add src/main/java/io/github/thebusybiscuit/slimefun4/libraries/dough/config/Config.java
git commit -m "refactor: update Config wrapper to use sudo"
```

---

### Task 5: Update Vendored CustomItemStack.java Wrapper

**Files:**
- Modify: `src/main/java/io/github/thebusybiscuit/slimefun4/libraries/dough/items/CustomItemStack.java`

**Interfaces:**
- Consumes: Task 3 (imports already renamed via sed)
- Produces: CustomItemStack wrapper delegates to SuDo

- [ ] **Step 1: Verify sed renamed all references**

The sed `s/dev\.yanianz\.star\./dev.yanianz.sudo./g` should have caught all `dev.yanianz.star.items.CustomItemStack` references. Verify:

```bash
grep "dev.yanianz.star" src/main/java/io/github/thebusybiscuit/slimefun4/libraries/dough/items/CustomItemStack.java
```
Expected: No output (no remaining star references)

- [ ] **Step 2: Commit**

```bash
git add src/main/java/io/github/thebusybiscuit/slimefun4/libraries/dough/items/CustomItemStack.java
git commit -m "refactor: update CustomItemStack wrapper to use sudo"
```

---

### Task 6: Update Vendored CustomGameProfile.java Wrapper

**Files:**
- Modify: `src/main/java/io/github/thebusybiscuit/slimefun4/libraries/dough/skins/CustomGameProfile.java`

**Interfaces:**
- Consumes: Task 3
- Produces: CustomGameProfile wrapper uses SuDo's PlayerHeadAdapter

- [ ] **Step 1: Verify import was renamed**

```bash
grep "dev.yanianz.sudo.skins.nms.PlayerHeadAdapter" src/main/java/io/github/thebusybiscuit/slimefun4/libraries/dough/skins/CustomGameProfile.java
```
Expected: Match on the import line.

- [ ] **Step 2: Commit**

```bash
git add src/main/java/io/github/thebusybiscuit/slimefun4/libraries/dough/skins/CustomGameProfile.java
git commit -m "refactor: update CustomGameProfile wrapper to use sudo"
```

---

### Task 7: Adapt Vendored PlayerHead.java Wrapper

**Files:**
- Modify: `src/main/java/io/github/thebusybiscuit/slimefun4/libraries/dough/skins/PlayerHead.java`

**Interfaces:**
- Consumes: Task 3 (imports renamed), Task 2 (PlayerSkin import renamed)
- Produces: PlayerHead uses SuDo's `skin.toProfile("CS-CoreLib").apply(meta)` instead of reflection

- [ ] **Step 1: Replace getItemStack(PlayerSkin) method body**

Replace lines 30-40 (the reflection-based apply):
```java
    @Nonnull
    public static ItemStack getItemStack(@Nonnull PlayerSkin skin) {
        return getItemStack(meta -> {
            try {
                java.lang.reflect.Method apply = skin.getProfile().getClass().getDeclaredMethod("apply", SkullMeta.class);
                apply.setAccessible(true);
                apply.invoke(skin.getProfile(), meta);
            } catch (ReflectiveOperationException ex) {
                ex.printStackTrace();
            }
        });
    }
```
With:
```java
    @Nonnull
    public static ItemStack getItemStack(@Nonnull PlayerSkin skin) {
        return getItemStack(meta -> {
            try {
                skin.toProfile("CS-CoreLib").apply(meta);
            } catch (ReflectiveOperationException ex) {
                ex.printStackTrace();
            }
        });
    }
```

- [ ] **Step 2: Replace setSkin method body**

Replace lines 51-69 (the reflection-based getGameProfile):
```java
    public static void setSkin(@Nonnull Block block, @Nonnull PlayerSkin skin, boolean updatePlayers) {
        if (adapter == null) {
            throw new UnsupportedOperationException("Cannot update skin texture, no adapter found");
        }

        Material type = block.getType();
        if (type != Material.PLAYER_HEAD && type != Material.PLAYER_WALL_HEAD) {
            throw new IllegalArgumentException("Block is not a player head: " + type);
        }

        try {
            Object customProfile = skin.getProfile();
            java.lang.reflect.Method getGameProfile = customProfile.getClass().getMethod("getGameProfile");
            GameProfile gameProfile = (GameProfile) getGameProfile.invoke(customProfile);
            adapter.setGameProfile(block, gameProfile, updatePlayers);
        } catch (ReflectiveOperationException ex) {
            ex.printStackTrace();
        }
    }
```
With:
```java
    public static void setSkin(@Nonnull Block block, @Nonnull PlayerSkin skin, boolean updatePlayers) {
        if (adapter == null) {
            throw new UnsupportedOperationException("Cannot update skin texture, no adapter found");
        }

        Material type = block.getType();
        if (type != Material.PLAYER_HEAD && type != Material.PLAYER_WALL_HEAD) {
            throw new IllegalArgumentException("Block is not a player head: " + type);
        }

        try {
            GameProfile gameProfile = skin.toProfile("CS-CoreLib");
            adapter.setGameProfile(block, gameProfile, updatePlayers);
        } catch (ReflectiveOperationException ex) {
            ex.printStackTrace();
        }
    }
```

- [ ] **Step 3: Remove unused import (if no longer needed)**

The reflection import `java.lang.reflect.Method` and `java.lang.reflect.InvocationTargetException` and `InstantiationException` may become unused. Check and remove.

- [ ] **Step 4: Commit**

```bash
git add src/main/java/io/github/thebusybiscuit/slimefun4/libraries/dough/skins/PlayerHead.java
git commit -m "refactor: adapt PlayerHead wrapper to use sudo toProfile() API"
```

---

### Task 8: Create Local ChatColors Replacement Utility

**Files:**
- Modify: `src/main/java/io/github/thebusybiscuit/slimefun4/libraries/dough/common/ChatColors.java`

**Interfaces:**
- Produces: Local ChatColors utility using Adventure APIs (no Star/Dough dependency)
- Consumed by: Task 9 (call site updates)

**Background:** `ChatColors.color()` is used in ~75 lines across 30 files, primarily returning a `String` with `§` codes for use in `sender.sendMessage()`, item meta, and string comparison. Converting to `MiniMessages.component()` (returns `Component`) would require changing every call site from String to Component context. Instead, we create a local utility that preserves string output but replaces the Star dependency with Adventure `LegacyComponentSerializer`.

- [ ] **Step 1: Write the replacement ChatColors.java**

Replace the entire file content with:
```java
package io.github.thebusybiscuit.slimefun4.libraries.dough.common;

import javax.annotation.Nonnull;
import javax.annotation.ParametersAreNonnullByDefault;

import org.bukkit.ChatColor;

import net.kyori.adventure.text.Component;
import net.kyori.adventure.text.serializer.legacy.LegacyComponentSerializer;

public final class ChatColors {

    private static final LegacyComponentSerializer SERIALIZER = LegacyComponentSerializer.legacyAmpersand();

    private ChatColors() {}

    /**
     * Translates alternate color codes ('&') to legacy section-sign ('§') codes.
     * Use {@link #component(String)} for Adventure Component output instead.
     */
    @Nonnull
    public static String color(@Nonnull String input) {
        return ChatColor.translateAlternateColorCodes('&', input);
    }

    /**
     * Translates alternate color codes to an Adventure Component.
     */
    @Nonnull
    public static Component component(@Nonnull String input) {
        return SERIALIZER.deserialize(input);
    }

    /**
     * Applies alternating ChatColors per character, returning a legacy string.
     */
    @Nonnull
    @ParametersAreNonnullByDefault
    public static String alternating(@Nonnull String text, ChatColor... colors) {
        StringBuilder builder = new StringBuilder();

        for (int i = 0; i < text.length(); i++) {
            builder.append(colors[i % colors.length]);
            builder.append(text.charAt(i));
        }

        return builder.toString();
    }
}
```

- [ ] **Step 2: Commit**

```bash
git add src/main/java/io/github/thebusybiscuit/slimefun4/libraries/dough/common/ChatColors.java
git commit -m "refactor: replace star ChatColors with local Adventure-based utility"
```

---

### Task 9: Update ChatColors Call Sites to Use MiniMessages Where Applicable

**Files:**
- Modify: ~30 files with ChatColors import + usages

**Background:** The import `import dev.yanianz.star.common.ChatColors` was NOT renamed in Task 3 (it was excluded since ChatColors has no SuDo equivalent). Now we need to update all 30 files to import from the vendored wrapper: `import io.github.thebusybiscuit.slimefun4.libraries.dough.common.ChatColors`.

- [ ] **Step 1: Replace all ChatColors imports**

```bash
find src -name "*.java" -exec sed -i '' 's/import dev\.yanianz\.star\.common\.ChatColors;/import io.github.thebusybiscuit.slimefun4.libraries.dough.common.ChatColors;/g' {} +
```

- [ ] **Step 2: Generate the import statement in MiniMessages.java**

Add import to any file that needs `MiniMessages`:

Files that use `ChatColors.color()` for Adventure-aware contexts (sendMessage, displayName, lore) will also need `import static dev.yanianz.sudo.common.MiniMessages.component;` or `import dev.yanianz.sudo.common.MiniMessages;` as needed. However, since we're keeping the local ChatColors utility that returns String, no additional imports are needed at this time. The existing call sites continue to work with the local ChatColors utility.

- [ ] **Step 3: Commit**

```bash
git add src/
git commit -m "fix: update ChatColors imports to local vendored utility"
```

---

### Task 10: Add Missing SuDo Import for slimefun4.libraries.dough.skins Subpackage

**Files:**
- Modify: Files in `src/main/java/io/github/thebusybiscuit/slimefun4/libraries/dough/` that now use `dev.yanianz.sudo.*`

**Background:** The vendored wrapper files now import from `dev.yanianz.sudo.*` (after Task 2 & 3 renames). These files are in a subpackage of slimefun4 but reference relocated sudo classes. The relocate in build.gradle.kts handles this at build time — no additional work needed as long as imports are correct.

- [ ] **Step 1: Verify no stale references**

```bash
grep -r "dev.yanianz.star" src/main/java/io/github/thebusybiscuit/slimefun4/libraries/ && echo "STALE REFERENCES FOUND" || echo "Clean"
```
Expected: `Clean`

- [ ] **Step 2: Verify no stale dough references (except ChatColors)**

```bash
grep -r "io.github.bakedlibs.dough" src/main/java/io/github/thebusybiscuit/slimefun4/libraries/ && echo "STALE REFERENCES FOUND" || echo "Clean"
```
Expected: `Clean`

- [ ] **Step 3: Commit if changes needed**

```bash
git add src/main/java/io/github/thebusybiscuit/slimefun4/libraries/
git commit -m "chore: verify no stale star/dough references in vendored wrappers"
```

---

### Task 11: Build and Verify Compilation

**Files:**
- N/A (verification only)

- [ ] **Step 1: Clean and compile**

```bash
./gradlew clean compileJava
```
Expected: BUILD SUCCESSFUL — no compilation errors.

If errors occur:
1. Check for any missed import renames (`grep -r "dev.yanianz.star" src/`)
2. Check for any missed dough imports (`grep -r "io.github.bakedlibs.dough" src/`)
3. Fix and retry

- [ ] **Step 2: Run tests**

```bash
./gradlew test
```
Expected: BUILD SUCCESSFUL — all tests pass.

- [ ] **Step 3: Commit**

```bash
git commit -m "build: verified compilation and tests pass" --allow-empty
```

---

### Task 12: Final Cleanup — Remove All Stale References

**Files:**
- Verify: All source files

- [ ] **Step 1: Scan for any remaining star references**

```bash
grep -r "dev\.yanianz\.star" src/ --include="*.java"
```
Expected: No output.

- [ ] **Step 2: Scan for any remaining Dough references**

```bash
grep -r "io\.github\.bakedlibs\.dough" src/ --include="*.java"
```
Expected: No output (all dough imports replaced or removed).

- [ ] **Step 3: Scan for any remaining ChatColors references from star**

```bash
grep -r "dev\.yanianz\.star\.common\.ChatColors" src/ --include="*.java"
```
Expected: No output.

- [ ] **Step 4: Verify shadowJar builds**

```bash
./gradlew shadowJar
```
Expected: BUILD SUCCESSFUL — fat JAR produced.

- [ ] **Step 5: Full build**

```bash
./gradlew build
```
Expected: BUILD SUCCESSFUL.

- [ ] **Step 6: Final commit**

```bash
git add -A
git commit -m "chore: final cleanup - no remaining star or dough references"
```
