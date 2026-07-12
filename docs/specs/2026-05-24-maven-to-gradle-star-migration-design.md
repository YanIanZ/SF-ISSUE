# Maven → Gradle KTS + Dough → Star Migration

**Date**: 2026-05-24
**Branch**: main

## §G — Goals

1. Replace Maven (`pom.xml`) with Gradle Kotlin DSL (`build.gradle.kts`)
2. Target Java 25 (`release = 25`)
3. Replace Dough (`com.github.Slimefun.dough:dough-api`) with Star (`com.github.YanIanZ:Star:v2.0.2`) from JitPack
4. Add credit "Maintained by iYanZ" on plugin enable

## §C — Constraints

- All existing Slimefun4 functionality must remain unchanged
- 272+ source files need `io.github.bakedlibs.dough.*` → `dev.yanianz.star.*` import migration
- 43 test files need same import migration
- Existing Paper API, third-party plugin integrations, and test framework must still work
- `jitpack.yml` must be updated for Gradle + Java 25

## §I — Interfaces / Components

### Build System

| File | Action |
|------|--------|
| `pom.xml` | **Delete** |
| `settings.gradle.kts` | **Create** — root project name `Slimefun` |
| `build.gradle.kts` | **Create** — monolithic build, all config |
| `gradle/wrapper/gradle-wrapper.properties` | **Create** — Gradle 8.13+ wrapper for Java 25 |
| `gradlew` | **Create** — Unix wrapper script |
| `gradlew.bat` | **Create** — Windows wrapper script |
| `Makefile` | Keep (standalone `gen-biomes` target) |
| `jitpack.yml` | **Update** — Java 25 + Gradle |

### Dependency Changes

| Dependency | Old | New |
|---|---|---|
| Dough / Star | `com.github.Slimefun.dough:dough-api:cb22e71335` | `com.github.YanIanZ:Star:v2.0.2` |
| Paper API | `io.papermc.paper:paper-api:1.21.1-R0.1-SNAPSHOT` | `io.papermc.paper:paper-api:1.21.11-R0.1-SNAPSHOT` (align with Star) |
| Mockito | `5.15.2` | `5.14.1` (align with Star) |
| MockBukkit | `com.github.seeseemelk:MockBukkit-v1.21:3.133.2` | `org.mockbukkit.mockbukkit:mockbukkit-v1.21:4.41.1` (align with Star) |

All other dependencies unchanged.

### Shade Relocation

| Original package | Shaded to |
|---|---|
| `dev.yanianz.star` | `io.github.thebusybiscuit.slimefun4.libraries.star` |
| `io.papermc.lib` | `io.github.thebusybiscuit.slimefun4.libraries.paperlib` |
| `org.apache.commons.lang` | `io.github.thebusybiscuit.slimefun4.libraries.commons.lang` |

### Source Code Changes

All imports `io.github.bakedlibs.dough.*` → `dev.yanianz.star.*`:

| Dough package | Star package |
|---|---|
| `dough.config` | `star.config` |
| `dough.items` | `star.items` |
| `dough.common` | `star.common` |
| `dough.skins` | `star.skins` |
| `dough.protection` | `star.protection` |
| `dough.blocks` | `star.blocks` |
| `dough.inventory` | `star.inventories` |
| `dough.collections` | `star.data` |
| `dough.scheduling` | `star.scheduling` |
| `dough.chat` | `star.chat` |
| `dough.recipes` | `star.recipes` |
| `dough.data.persistent` | `star.data.persistent` |

Note: `dough.collections` maps to `star.data` in Star's module structure. Individual class existence (e.g., `RandomizedSet`, `LoopIterator`) must be verified after JitPack resolution.

### Credits

- `src/main/java/.../implementation/Slimefun.java` — add `log(Level.INFO, "Maintained by iYanZ")` in `onPluginStart()` after localization setup
- `src/main/resources/plugin.yml` — update `author` field to `iYanZ`

## §V — Verification

1. `./gradlew build` compiles without errors
2. `./gradlew test` passes all existing tests
3. `./gradlew shadowJar` produces shaded JAR with Star classes relocated correctly
4. JAR loads as Paper plugin — no `ClassNotFoundException` for Star classes
5. Plugin enable shows "Maintained by iYanZ" in console log
6. All existing Slimefun4 features operate normally

## §T — Tasks

| ID | Task | Dependencies |
|----|------|-------------|
| T.1 | Create `settings.gradle.kts` and `build.gradle.kts` | — |
| T.2 | Set up Gradle wrapper (`gradlew`, `gradle-wrapper.properties`) | T.1 |
| T.3 | Add all dependencies (Paper, Star, test, third-party) | T.1 |
| T.4 | Configure Shadow plugin with Star relocation | T.3 |
| T.5 | Update `jitpack.yml` for Java 25 + Gradle | T.2 |
| T.6 | Delete `pom.xml` | T.4 |
| T.7 | Refactor all imports: `io.github.bakedlibs.dough` → `dev.yanianz.star` | T.3 |
| T.8 | Update `plugin.yml` author + `Slimefun.java` credits | T.1 |
| T.9 | Run `./gradlew build` and fix compilation errors | T.6, T.7, T.8 |
| T.10 | Run `./gradlew test` and fix test failures | T.9 |
| T.11 | Run `./gradlew shadowJar` and verify JAR output | T.10 |
| T.12 | Update `.github/workflows/` for Gradle (if CI exists) | T.2 |
| T.13 | Update `.gitignore` — add `.gradle/`, `build/`, remove `target/` | T.6 |
