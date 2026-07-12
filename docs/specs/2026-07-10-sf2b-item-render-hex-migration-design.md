# Slimefun4 SF2b — Item Render Bridge + SlimefunItems Hex Conversion

**Date:** 2026-07-10
**Status:** Draft — awaiting user review
**Repo:** `/Users/rheninxy/Client-Project/Slimefun4`
**Target:** Java 25, Paper API 26.2, SuDo `26.2.0`, Adventure MiniMessage
**Sub-project:** SF2b — second slice of "SF2: kill dough / SuDo 100% / no legacy"

## Context

Slimefun4 has ~1550 legacy `&`-code display literals across 57 files; **1086 are in one file**,
`implementation/SlimefunItems.java` (the item registry — every item's name/lore). Two item-render
paths coexist:

- **`SlimefunItemStack`** renders name/lore via `ChatColor.translateAlternateColorCodes('&', …)` →
  **legacy vanilla colors** (8 sites across its constructors).
- **dough `CustomItemStack`** already bridges `&` → hex → MiniMessage via
  `ChatColors.hexString(…)` → SuDo, rendering the **custom hex palette** (`ChatColors.HEX_TAG_MAP`,
  e.g. `&7`→`#8D99AE`, `&e`→`#F4A261`).

The user chose the **custom-palette modern reskin** for the item text. This slice switches the
`SlimefunItemStack` render path onto the same MiniMessage bridge (delivering the reskin to all
~1000 registry items) and converts the 1086 `SlimefunItems.java` literals to native
`<#hex>`/`<format>` MiniMessage. The remaining ~464 literals (56 other files) are SF2c; deleting
`ChatColors` + the bridge + BlockStorage read-normalization is SF2d.

## Goals

1. Switch `SlimefunItemStack`'s render sites from `ChatColor.translateAlternateColorCodes` to
   `LEGACY.serialize(MiniMessages.component(ChatColors.hexString(s)))` — item text uses the custom
   hex palette (rendered as `§x`-hex strings), renders **both** legacy `&` and native `<#hex>`
   inputs, and keeps `setDisplayName(String)`/`setLore(List<String>)` so SF4's lore/name read-back
   (`ChatColor.stripColor`, `contains("<Type>")`, `replace("<ID>", …)`) keeps working.
2. Convert `SlimefunItems.java`'s 1086 `&`-code literals to native MiniMessage (`<#hex>` /
   `<bold>` / `<reset>` …) using ChatColors' `HEX_TAG_MAP` palette, with MiniMessage escaping.
3. `./gradlew build` green; all tests pass; items render the hex palette (manual spot-check).

## Non-Goals

- The other ~464 `&`-literals in 56 files (SF2c).
- Deleting `ChatColors` / the bridge / BlockStorage read-normalization (SF2d) — `ChatColors`
  stays as the render bridge this slice.
- `CustomItemStack` repoint to SuDo (SF2c) — the dough `CustomItemStack` bridge path is untouched.
- Any SuDo library change.

## Why the bridge decouples the change (key design point)

`ChatColors.hexString(s)` converts `&`/`§` codes to `<#hex>`/`<format>` tags using `CODE_PATTERN`,
which matches **only** `&`/`§` + code chars — it does **not** match `<#RRGGBB>` MiniMessage tags.
So `hexString` is idempotent on already-converted strings: `hexString("<#8D99AE>x") == "<#8D99AE>x"`.
Therefore `MiniMessages.component(ChatColors.hexString(s))` renders **both** a legacy `&7x` and a
native `<#8D99AE>x` identically. This lets the render switch (Goal 1) and the literal conversion
(Goal 2) be independent, and lets later slices convert the remaining literals incrementally
without breaking rendering. The visual reskin happens at Goal 1 (the bridge applies the palette to
the existing `&`-literals); Goal 2 is visually neutral (native strings render the same through the
bridge).

## Design

### 1. `SlimefunItemStack` render bridge (String-preserving — critical)

`api/items/SlimefunItemStack.java` has ~8 render sites (across its constructors) of the form:
```java
meta.setDisplayName(ChatColor.translateAlternateColorCodes('&', name));
// and, per lore line:
lines.add(ChatColor.translateAlternateColorCodes('&', line));
im.setLore(lines);
```

**Constraint discovered during planning — keep the String-based item meta.** SF4 heavily reads
item name/lore *back* as legacy strings and string-matches them: `ChatColor.stripColor(line)` +
`line.contains("<Type>")` + `line.replace("<Type>", …)` (`AbstractMonsterSpawner`), `.replace("<ID>", …)`
(`AbstractCraftingTable`, `BackpackListener`), and other permission/prefix checks on
`meta.getDisplayName()`/`getLore()`. Switching to Adventure `Component` setters
(`meta.displayName(Component)`/`meta.lore(List<Component>)`) would break all of this read-back
(and would parse the runtime placeholders `<Type>`/`<ID>` as MiniMessage tags). So the bridge MUST
keep `setDisplayName(String)`/`setLore(List<String>)`, producing the string by rendering through
MiniMessage (custom palette) and serializing **back to a legacy `§x`-hex string**:

```java
meta.setDisplayName(render(name));
// lore:
List<String> lines = new ArrayList<>();
for (String line : lore) {
    lines.add(render(line));
}
im.setLore(lines);
```
with a single private helper on `SlimefunItemStack`:
```java
private static final LegacyComponentSerializer LEGACY =
    LegacyComponentSerializer.builder().hexColors().build();

@Nonnull
private static String render(@Nonnull String s) {
    // ChatColors.hexString bridges legacy & / § codes to <#hex> tags (idempotent on
    // already-converted <#hex> strings); MiniMessages applies the custom hex palette;
    // LEGACY re-serializes to a §x-hex string so item meta stays String-based and SF4's
    // lore/name read-back (ChatColor.stripColor, contains("<Type>"), replace(...)) keeps working.
    return LEGACY.serialize(MiniMessages.component(ChatColors.hexString(s)));
}
```
- Imports added: `dev.yanianz.sudo.common.MiniMessages`,
  `io.github.thebusybiscuit.slimefun4.libraries.dough.common.ChatColors`,
  `net.kyori.adventure.text.serializer.legacy.LegacyComponentSerializer`. Keep `org.bukkit.ChatColor`
  only if still used elsewhere in the file.
- **Placeholder safety:** `<Type>`/`<ID>` are NOT MiniMessage codes, so `ChatColors.hexString`
  leaves them untouched; `MiniMessages.component(...)` renders any unrecognized `<Type>` tag as
  literal text (Adventure preserves unresolved tags), and `LEGACY.serialize` reproduces the literal
  `<Type>`/`<ID>` in the output string. The runtime `contains("<Type>")` / `replace("<Type>", …)`
  therefore still match. **Impl-verify** with a unit test: `render("&7Type: &b<Type>")` must yield a
  string containing the literal substring `<Type>` (and hex `§x` color codes), and
  `ChatColor.stripColor(render("&7Type: &b<Type>"))` must equal `"Type: <Type>"`.
- **Italic default:** the legacy-string path renders non-italic (item name italic is a client
  default on the *Component* path only); staying on `setDisplayName(String)` preserves the current
  non-italic look with no extra decoration handling.

### 2. Convert `SlimefunItems.java` (1086 literals)

Rewrite every `&`-code string literal in `implementation/SlimefunItems.java` to native MiniMessage,
using ChatColors' `HEX_TAG_MAP` palette. The mapping (verbatim from `ChatColors`):

| legacy | → | | legacy | → |
|---|---|---|---|---|
| `&0`/`§0` | `<#2B2D42>` | | `&8` | `<#4A4E69>` |
| `&1` | `<#1D3557>` | | `&9` | `<#219EBC>` |
| `&2` | `<#2A9D8F>` | | `&a` | `<#2A9D8F>` |
| `&3` | `<#457B9D>` | | `&b` | `<#8ECAE6>` |
| `&4` | `<#BC4742>` | | `&c` | `<#E76F51>` |
| `&5` | `<#9B5DE5>` | | `&d` | `<#CDB4DB>` |
| `&6` | `<#E9C46A>` | | `&e` | `<#F4A261>` |
| `&7` | `<#8D99AE>` | | `&f` | `<#F1FAEE>` |

Format codes: `&l`→`<bold>`, `&m`→`<strikethrough>`, `&n`→`<underline>`, `&o`→`<italic>`,
`&k`→`<obfuscated>`, `&r`→`<reset>`. The `&x&R&R&G&G&B&B` legacy-hex-expansion form →
`#RRGGBB` (per ChatColors' `convertMatch`).

**Converter requirements (the plan will pick the exact tool):**
- Codify ChatColors' `CODE_PATTERN` + `convertMatch` logic so the build-time conversion produces
  **byte-identical** output to what `ChatColors.hexString` would produce at runtime (guarantees
  visual parity). A small Java/JBang converter that reuses the ChatColors regex + map on each
  string literal is the safest; a per-code regex sweep is acceptable only if it reproduces the
  same output including the hex-expansion form.
- Operate on Java string literals only (the `"..."` args to `SlimefunItemStack`/`SlimefunItemStack`
  constructors). Preserve non-`&` content exactly.
- **MiniMessage escaping:** if a literal's text contains a raw `<` or `>` (rare in item names),
  escape it (`\<`) so MiniMessage does not misparse it. The converter must handle this.
- Idempotent: re-running the converter on already-converted literals is a no-op.
- After conversion, `grep '"[^"]*&[0-9a-fk-orA-FK-OR]' SlimefunItems.java` → no legacy `&`-code
  literals remain in that file (allowing for genuine `&` in prose, which the converter leaves and
  which the bridge renders literally).

### 3. Verification

- `./gradlew build` — compile + all existing tests (MockBukkit).
- Converter unit test: feed representative `&`-strings (`"&7Cargo Node &c(Connector)"`,
  `"&8⇨ &eMust be placed &a3 Blocks"`, a `&x…` hex-expansion sample, a string with a literal
  `<`) → assert the exact expected MiniMessage output (matching `ChatColors.hexString`).
- Manual visual spot-check: build the plugin, load on a test server, open the Slimefun guide, and
  confirm several converted items (e.g. `CARGO_CONNECTOR_NODE`, `REACTOR_ACCESS_PORT`) render with
  the custom hex palette and no raw `<#…>` tags or stray `&` codes.

## Success Criteria

1. `SlimefunItemStack` renders via `MiniMessages.component(ChatColors.hexString(…))` (no
   `translateAlternateColorCodes` remains in that file); items show the custom hex palette.
2. `SlimefunItems.java` contains zero legacy `&`-code display literals; its 1086 literals are
   native MiniMessage and render identically through the bridge.
3. `./gradlew build` green; all tests pass; converter unit test passes; manual spot-check clean.
4. No other file's literals changed (SF2c scope); `ChatColors` + the dough `CustomItemStack`
   bridge remain (SF2d scope).

## Risks

- **Visual reskin scope:** Goal 1 changes the color of ~1000 item names/lore from vanilla to the
  custom palette. This is intended (the user chose the reskin). Spot-check confirms it looks right.
- **Italic default:** Adventure item text defaults to italic; the design sets ITALIC=false to
  match the legacy look. Impl must verify parity with SuDo's item setters and not double-toggle.
- **Converter correctness on 1086 literals:** mitigated by mirroring ChatColors' exact logic +
  a unit test + the idempotency/escaping requirements + the build as the gate. A literal the
  converter mangles would fail compile (broken string) or show as a raw tag in the spot-check.
- **Non-item `&`-literals in scope creep:** the converter targets `SlimefunItems.java` only; other
  files are SF2c. The bridge renders any still-`&` literal elsewhere correctly, so nothing breaks.
- **Lore/name read-back (the planning-discovered landmine):** SF4 reads item name/lore back as
  legacy strings and string-matches (`<Type>`/`<ID>` placeholders, `ChatColor.stripColor`,
  permission/prefix checks). Mitigated by keeping the item meta **String-based** (§x-hex via
  `LegacyComponentSerializer`, not Adventure `Component` setters) — see §1. A unit test asserts
  `<Type>` survives `render(...)` and `stripColor(render(...))` yields the expected plain text.
