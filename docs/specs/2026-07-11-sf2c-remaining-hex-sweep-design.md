# Slimefun4 SF2c — Remaining `&`-Literal Hex Sweep

**Date:** 2026-07-11
**Status:** Draft — awaiting user review
**Repo:** `/Users/rheninxy/Client-Project/Slimefun4`
**Target:** Java 25, Paper API 26.2, SuDo `26.2.0`
**Sub-project:** SF2c — third slice of "SF2: kill dough / SuDo 100% / no legacy"

## Context

SF2b converted `SlimefunItems.java`'s 1086 legacy `&`-code literals to native `<#hex>` MiniMessage
and made `SlimefunItemStack` render the custom hex palette. **466 `&`-code literals remain across
57 other files** (item classes, GUI/menu helpers, messages, holograms, misc). This slice converts
all of them, continuing toward the end state: all-hex, no `org.bukkit.ChatColor`, no
`libraries/dough`. `ChatColor`-API removal (190 refs), `CustomItemStack`→SuDo (65 sites), and
killing `ChatColors`/`dough` are the following slices (SF2d/SF2e/SF2f).

## Goals

1. Convert all 466 remaining legacy `&`-code literals in `src/main` (outside `SlimefunItems.java`
   and `libraries/dough/`) to native `<#hex>`/`<format>` MiniMessage, using ChatColors' palette.
2. Keep rendering correct: the sinks these literals feed already bridge `&`/`<#hex>` to MiniMessage
   (verified below), so the conversion is visually neutral. Route any raw sink that does not
   through MiniMessage (verified: none currently carry `&`-literals — see Risks).
3. Read-back audit: normalize any lore/name string comparisons the conversion touches (the SF2b
   landmine), so nothing that reads item/GUI text back and matches against a constant breaks.
4. `./gradlew build` green; all tests pass; spot-check representative files.

## Non-Goals

- `org.bukkit.ChatColor` API usage (190 refs: `ChatColor.GREEN + "…"` constants, `ChatColor.stripColor`) → SF2d.
- `CustomItemStack` (dough) repoint to SuDo (65 sites) → SF2e.
- Killing `ChatColors` + deleting/rebranding `libraries/dough` → SF2f.
- Any SuDo library change.

## Verified sink situation (why the sweep is safe)

The 466 literals feed sinks that already render MiniMessage / bridge legacy:
- **Localization** (`SlimefunLocalization.sendMessage`) sends via `MiniMessages.component(prefix + getMessage(...))`.
- **`CustomItemStack.create`** (dough) bridges via `legacyToMiniMessage` = `ChatColors.hexString` → MiniMessage.
- **`SlimefunItemStack`** renders via the SF2b `render()` bridge (`§x`-hex).
- **`ChatColors.color(...)`** (41 `sendMessage(ChatColors.color(...))` sites) returns a rendered `Component`.
- **`ChatColors.hexString(...)`** used directly for `setDisplayName`/`setLore` (idempotent bridge).
- **Holograms** — `setHologramLabel(label)` receives labels already bridged by callers (e.g.
  `HologramOwner` passes `ChatColors.hexString(text)`).
- **Raw `sendMessage("&…")` with a legacy literal: 0 sites** (verified) — so no raw-legacy sink
  carries a `&`-literal that a blanket sweep would break.

`ChatColors.hexString` is idempotent on already-`<#hex>` input (its `CODE_PATTERN` matches only
`&`/`§` codes), so a swept literal renders identically through every bridge above.

## Design

### 1. The sweep (deterministic, palette-exact)

Apply the SF2b palette conversion to every legacy `&`-code in the 57 target files (all `src/main`
`.java` except `SlimefunItems.java` and anything under `libraries/dough/`). Palette (ChatColors'
`HEX_TAG_MAP`): `&0`→`<#2B2D42>` … `&f`→`<#F1FAEE>` (16 colors), `&l`→`<bold>`, `&m`→`<strikethrough>`,
`&n`→`<underline>`, `&o`→`<italic>`, `&k`→`<obfuscated>`, `&r`→`<reset>`, and the `&x&R&R&G&G&B&B`
hex-expansion form → `#RRGGBB`.

**Per-file safety (unlike SlimefunItems, these files contain Java logic, not just literals):**
- A blanket per-code `perl s/&7/<#8D99AE>/g` is UNSAFE here — `&` also appears as Java operators
  (`&&`, bitwise `&`) and in non-display strings. The sweep must convert `&`-codes **only inside
  string literals**. The plan uses a converter that (a) matches double-quoted string literals and
  (b) applies the code→tag map within each literal, leaving Java code untouched. It also escapes a
  literal `<`/`>` that is NOT a runtime placeholder, and preserves runtime placeholders
  (`<Type>`, `<ID>`, `%usage%`, etc.) and `& `-in-prose.
- The plan verifies preconditions per batch (no unintended `&&`/bitwise hits; placeholder
  inventory) before applying, and diffs a sample from each file.

### 2. Sink verification (no raw-legacy break)

After the sweep, confirm no converted literal reaches a raw legacy sink:
- `grep` for `sendMessage("<#` / `sendMessage(".*<#` (a hex literal sent to the String overload) →
  expect none; if any appear, wrap in `MiniMessages.component(...)`.
- `grep` for `setLore`/`setDisplayName` receiving a bare swept literal not via `CustomItemStack`/
  `ChatColors`/`render` → route through `ChatColors.hexString`/`MiniMessages` as the surrounding
  code already does.

### 3. Read-back audit (the SF2b landmine, scaled)

For the touched files, find lore/name comparisons against hardcoded color constants or `hexString`
needles and make them format-robust (normalize both sides via `ChatColors.hexString`, or compare
`ChatColor.stripColor(...)` text) — the pattern established in SF2b (`AbstractCraftingTable`,
`BackpackListener`, `PlayerProfile.getBackpack`, `LimitedUseItem`). Carry the SF2b deferrals:
`AbstractEnchantmentMachine:84` normalization asymmetry; unify the `hexString("&7ID:…")` needle
literals.

## Testing

- `./gradlew build` — compile + all existing tests (MockBukkit).
- Grep: zero legacy `&`-codes remain in the 57 target files; placeholders + `& `-prose intact.
- Spot-check a sample per sink type: a CustomItemStack GUI button (ChestMenuUtils), a localized
  message path, an item class, a hologram caller — confirm the swept literal renders (no raw
  `<#…>` tags).
- Any test that asserts old `§`/`ChatColor`-format lore/name is updated to compare stripped/
  normalized text (as `TestBackpackCommand` was in SF2b).

## Success Criteria

1. Zero legacy `&`-codes remain in `src/main` outside `libraries/dough/` (SlimefunItems already done).
2. No raw-legacy sink carries a `<#hex>` literal (grep clean); all display renders correctly.
3. Lore/name read-back comparisons in touched files are format-robust; no regression.
4. `./gradlew build` green; all tests pass.
5. `libraries/dough` still holds only `ChatColors` + `CustomItemStack` (SF2f removes them).

## Risks

- **Read-back landmine at scale:** more files means more hardcoded-color comparisons that the reskin
  can break silently (runtime, not compile). Mitigation: the §3 audit + build/tests, plus the
  standing RUNTIME-QA caveat — verify on a live server before release. This is the dominant risk.
- **Sweep touching non-display strings:** mitigated by literal-scoped conversion + per-batch
  precondition checks + sample diffs; a mangled literal fails compile or shows raw tags in the
  spot-check.
- **Heterogeneous sinks:** verified all bridge MiniMessage and 0 raw-legacy-literal sinks exist;
  §2 re-verifies post-sweep.
