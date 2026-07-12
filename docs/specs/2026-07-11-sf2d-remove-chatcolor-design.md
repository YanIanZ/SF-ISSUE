# Slimefun4 SF2d — Remove `org.bukkit.ChatColor` Entirely

**Date:** 2026-07-11
**Status:** Draft — awaiting user review
**Repo:** `/Users/rheninxy/Client-Project/Slimefun4`
**Target:** Java 25, Paper API 26.2, SuDo `26.2.0`
**Sub-project:** SF2d — fourth slice of "SF2: kill dough / SuDo 100% / no legacy"

## Context

SF2b/SF2c converted all legacy `&`-code string literals to native `<#hex>` MiniMessage. The other
legacy-color form remains: **`org.bukkit.ChatColor` API — 188 refs across 42 files** (color
constants, `stripColor`, type-usages, misc). This slice removes it entirely, converting to the
custom hex palette (consistent with SF2b/c), using inline `<#hex>` tags (no new helper) and SuDo's
`MiniMessages` for stripping.

## Goals

1. Zero `org.bukkit.ChatColor` references remain in `src/main` (outside `libraries/dough/`); the
   import is removed from all 42 files.
2. Color constants → inline `<#hex>` tags per the ChatColors `CHATCOLOR_HEX_MAP` palette.
3. `ChatColor.stripColor(s)` → `MiniMessages.plain(MiniMessages.stripLegacy(s))` (handles both
   `§x`-hex item lore and `<#hex>` MiniMessage → plain).
4. `ChatColor`-typed fields/params/locals → `String` (hex tag).
5. Rendering stays correct (sink-route `<#hex>` results to MiniMessage sinks) and lore/name
   read-back stays robust.
6. `./gradlew build` green; all tests pass.

## Non-Goals

- `CustomItemStack` (dough) → SuDo (SF2e).
- Killing `ChatColors` + deleting/rebranding `libraries/dough` (SF2f).
- Any SuDo library change (`MiniMessages.stripLegacy`/`plain` already exist).

## The palette (ChatColors `CHATCOLOR_HEX_MAP`, verbatim)

| ChatColor | → hex tag | | ChatColor | → hex tag |
|---|---|---|---|---|
| `BLACK` | `<#2B2D42>` | | `DARK_GRAY` | `<#4A4E69>` |
| `DARK_BLUE` | `<#1D3557>` | | `BLUE` | `<#219EBC>` |
| `DARK_GREEN` | `<#2A9D8F>` | | `GREEN` | `<#2A9D8F>` |
| `DARK_AQUA` | `<#457B9D>` | | `AQUA` | `<#8ECAE6>` |
| `DARK_RED` | `<#BC4742>` | | `RED` | `<#E76F51>` |
| `DARK_PURPLE` | `<#9B5DE5>` | | `LIGHT_PURPLE` | `<#CDB4DB>` |
| `GOLD` | `<#E9C46A>` | | `YELLOW` | `<#F4A261>` |
| `GRAY` | `<#8D99AE>` | | `WHITE` | `<#F1FAEE>` |

Format/misc: `RESET` → `<reset>`; `COLOR_CHAR` → the literal char `'§'` (0x00A7); the single
`translateAlternateColorCodes('&', s)` → `MiniMessages.component(s)` (routed per §5).

## Design

### 1. Color constants (~165 refs)

Replace each `ChatColor.<NAME>` with its inline `<#hex>` tag (table above). Cases:
- **`ChatColor.X + "text"` concat (45 sites)** → `"<#hex>text"`. Then the result must reach a
  MiniMessage-rendering sink. Most already do (localization `MiniMessages.component`,
  `CustomItemStack`/`ChatColors.hexString`, `SlimefunItemStack.render`). For a raw legacy sink
  (`sender.sendMessage(String)`, `meta.setDisplayName(String)`, `setLore(List<String>)` not via a
  bridge), wrap the assembled string in `MiniMessages.component(...)` (import
  `dev.yanianz.sudo.common.MiniMessages`) — the same fix SF2c applied to `CargoManager`. §5 audits
  these.
- **Constants inside a MiniMessage/`ChatColors` string** (e.g. concatenated then passed to
  `ChatColors.color(...)`/`hexString(...)`) → `<#hex>` inline; the bridge renders it.
- **Regex/`Pattern` built from a constant** (e.g. `PatternUtils`) → build from the `<#hex>` string
  and match against a normalized/stripped line (see §4).

### 2. `ChatColor.stripColor` (19 sites)

`ChatColor.stripColor(s)` → `MiniMessages.plain(MiniMessages.stripLegacy(s))`.
- `stripLegacy` removes `§`/`§x`-hex codes; `plain` removes any `<#hex>` MiniMessage tags → plain
  text for both item lore (`§x`-hex) and MiniMessage strings.
- Where `stripColor` output is compared to a constant, the constant becomes its plain form.

### 3. `ChatColor`-typed fields / params / locals (4 files, 10 refs)

- `core/attributes/Radioactivity.java` — enum field `ChatColor color` + ctor param → `String color`
  (store the `<#hex>` tag); update the enum constructor arguments.
- `core/services/profiler/PerformanceRating.java` — same shape (enum `ChatColor color` → `String`).
- `core/commands/subcommands/VersionsCommand.java` — locals `ChatColor primaryColor/secondaryColor`
  → `String` hex tags; update the branches that assign them and their concatenation use.
- `utils/ChatUtils.java` — `crop(ChatColor color, String)` → `crop(String color, String)` (hex
  tag); update its call sites (grep `ChatUtils.crop(`).

### 4. Read-back + sink audit (the SF2b/c landmine)

- Any comparison of item/GUI lore or display name against a constant that was `ChatColor.X + "…"`
  now becomes `"<#hex>…"`; make the comparison format-robust — normalize both sides via
  `ChatColors.hexString(...)` or compare `MiniMessages.plain(MiniMessages.stripLegacy(...))` text
  (the established pattern). Do NOT rewrite self-consistent set-and-read pairs (e.g.
  `SOULBOUND_LORE`, `FireworksListener`) beyond the constant substitution — keep both sides using
  the same new constant so they still match.
- Sink audit: `grep` for `<#hex>` literals reaching raw legacy sinks after conversion
  (`sendMessage("…<#`, `setDisplayName("…<#`, `setLore(…<#`, and — the SF2c lesson — `<#hex>`
  nested inside `translateAlternateColorCodes(...)`); wrap each in `MiniMessages.component(...)`.

### 5. Cleanup

Remove `import org.bukkit.ChatColor;` from every file once its last reference is gone. Final grep:
`grep -rn "org.bukkit.ChatColor\|ChatColor\." src/main | grep -v "/libraries/dough/"` → no output
(dough's `ChatColors` is a different class, kept until SF2f).

## Testing

- `./gradlew build` — compile + all existing tests (MockBukkit).
- Grep: zero `org.bukkit.ChatColor` refs / imports in `src/main` outside `libraries/dough/`.
- Update any test asserting old `ChatColor`/`§`-format text to compare stripped/normalized text
  (as `TestBackpackCommand` in SF2b) — assert content, not color bytes.
- Spot-check: a converted `ChatColor.X + msg` message renders colored (not raw tags); a
  `stripColor`→`plain(stripLegacy)` read yields the same plain text as before; the refactored
  enum-color usages (`Radioactivity`, `PerformanceRating`, `VersionsCommand`) render correctly.

## Success Criteria

1. Zero `org.bukkit.ChatColor` references + imports in `src/main` outside `libraries/dough/`.
2. Constants render via the custom hex palette; `stripColor` replaced by `plain(stripLegacy)`;
   type-usages refactored to `String` hex tags.
3. No `<#hex>` literal reaches a raw legacy sink; lore/name read-back robust.
4. `./gradlew build` green; all tests pass.
5. `libraries/dough` (ChatColors + CustomItemStack) untouched (SF2e/f).

## Risks

- **Biggest slice yet — 188 refs / 42 files, heterogeneous shapes.** Not a clean mechanical sweep:
  constants (sink-routing), `stripColor` (read-back), type-refactors (structural). Mitigation:
  per-shape handling, the §4 audit, build/tests, and the standing **RUNTIME-QA caveat** (verify on
  a live server — many display/read-back paths are untested).
- **`plain(stripLegacy)` vs bare `stripColor` edge cases:** for a plain string with no codes both
  return it unchanged; for `§x`-hex both strip to plain; for `<#hex>` only the new form strips the
  tags (old `stripColor` left them). This is an improvement, not a regression, but spot-check the
  comparison sites that feed off `stripColor` output.
- **Type-refactor ripple:** changing a `ChatColor` field/param to `String` touches its constructor
  args and call sites — the plan enumerates each; the compiler catches misses.
