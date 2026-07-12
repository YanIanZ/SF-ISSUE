# Slimefun4 SF2a — dough Repoint (Config + dead skins shims)

**Date:** 2026-07-10
**Status:** Draft — awaiting user review
**Repo:** `/Users/rheninxy/Client-Project/Slimefun4`
**Target:** Java 25, Paper API 26.2, SuDo `26.2.0`
**Sub-project:** SF2a — first slice of "SF2: kill dough / SuDo 100%"

## Context

Slimefun4 vendors a shrunken `libraries/dough/` package (5 shim classes over the SuDo library).
The goal of SF2 is to eliminate the `dough` namespace and reach 100% SuDo. During brainstorming
the full effort was decomposed into slices because killing the legacy `&`-code layer (`ChatColors`)
requires converting ~1550 legacy string literals + internal legacy sources (`LoreBuilder`,
`BlockStorage`-stored names, item-meta) across 57 files — a large multi-slice sub-epic:

- **SF2a (this spec)** — repoint the SuDo-equivalent classes with no legacy coupling.
- **SF2b** — legacy `&`/`§` → MiniMessage converter + sweep the 1550 literals + `LoreBuilder`.
- **SF2c** — repoint `CustomItemStack` to SuDo (inputs are MiniMessage after SF2b).
- **SF2d** — kill `ChatColors` (convert its 61 sites + BlockStorage read-normalization) + delete
  `libraries/dough` entirely.

SF2a is bounded and mechanical: three shim files with clean SuDo equivalents and no `&`-code
involvement.

## Goals

1. Repoint all callers of `libraries.dough.config.Config` to `dev.yanianz.sudo.config.Config`.
2. Delete the three dough shim files that are now redundant: `config/Config.java`,
   `skins/PlayerHead.java`, `skins/CustomGameProfile.java`.
3. `./gradlew build` compiles and all existing tests pass.

## Non-Goals

- `CustomItemStack` (SF2c — coupled to the `&`→MiniMessage conversion).
- `ChatColors` and the 1550-literal legacy purge (SF2b/SF2d).
- Any SuDo library change — SuDo `26.2.0` is consumed as-is.

## Verified state (from Slimefun4 `src/main`)

- **`dough.config.Config`** is a pure subclass: `public class Config extends
  dev.yanianz.sudo.config.Config` whose five constructors each call only `super(...)`. SuDo's
  `Config` has the five identical constructors (`Plugin`; `Plugin,String`;
  `File,FileConfiguration`; `File`; `String`). The subclass adds nothing. **19 files** import
  `io.github.thebusybiscuit.slimefun4.libraries.dough.config.Config`.
- **`dough.skins.PlayerHead`** and **`dough.skins.CustomGameProfile`** have **zero importers** —
  no file references `libraries.dough.skins.*`. The four `PlayerHead.getItemStack`/`setSkin` call
  sites (`SlimefunUtils`, `ProgrammableAndroid`, `DebugFishListener`, `SlimefunItemStack`) already
  import `dev.yanianz.sudo.skins.PlayerHead` directly. So both dough skins files are dead code.

## Design

### 1. Config repoint (19 files)

For every file importing the dough Config, replace the import line:

```java
import io.github.thebusybiscuit.slimefun4.libraries.dough.config.Config;
```
→
```java
import dev.yanianz.sudo.config.Config;
```

No other change: the type is used identically (the dough class IS-A `dev.yanianz.sudo.config.Config`
and overrides nothing), so every `new Config(...)` and method call resolves to the SuDo class with
the same signature. **Verification step:** confirm no code depends on the dough *subtype*
specifically (`grep` for `dough.config.Config` used in an `instanceof`, a cast, or a
method/field declared with the dough type) before deleting — expected none, since the subclass
adds no members.

The exact file set is the 19 that import the dough Config (enumerated in the plan by grep). If one
call site references the type without an `import` (fully-qualified `...libraries.dough.config.Config`),
rewrite that fully-qualified reference too.

### 2. Delete the three shim files

- `git rm src/main/java/io/github/thebusybiscuit/slimefun4/libraries/dough/config/Config.java`
  (after the 19 repoints, it has no importers).
- `git rm .../libraries/dough/skins/PlayerHead.java` (dead — 0 importers).
- `git rm .../libraries/dough/skins/CustomGameProfile.java` (dead — 0 importers).

After SF2a, `libraries/dough/` retains only `common/ChatColors.java` and
`items/CustomItemStack.java` (deferred to SF2b–d).

## Testing

- `./gradlew build` — compile + all existing tests (MockBukkit). The Config change is a pure
  import swap between a subclass and its identical-API superclass; the shim deletions remove dead
  code. No behavior change, so existing tests are the gate.

## Success Criteria

1. Zero references to `libraries.dough.config.Config`, `libraries.dough.skins.PlayerHead`,
   `libraries.dough.skins.CustomGameProfile` remain in `src/main` (`grep` clean).
2. The three shim files are deleted.
3. `./gradlew build` is green; all tests pass.
4. `libraries/dough/` contains only `common/ChatColors.java` and `items/CustomItemStack.java`.

## Risks

- **Subtype dependency:** if any code relied on the dough `Config` being a distinct type, the swap
  would break. Mitigation: the subclass overrides nothing and adds no members, and the design
  verifies no `instanceof`/cast/declaration uses the dough type before deleting. The build is the
  final gate.
- **Config-version / behavior parity:** SuDo `Config` is the actual implementation the dough class
  already extended, so runtime behavior is unchanged by definition.
