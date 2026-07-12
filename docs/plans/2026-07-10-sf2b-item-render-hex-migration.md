# Slimefun4 SF2b — Item Render Bridge + SlimefunItems Hex Conversion Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Render `SlimefunItemStack` item text through the custom hex palette (via a MiniMessage bridge serialized back to `§x`-hex strings, preserving lore read-back) and convert `SlimefunItems.java`'s 1086 legacy `&`-code literals to native MiniMessage.

**Architecture:** `SlimefunItemStack`'s render helper becomes `LEGACY.serialize(MiniMessages.component(ChatColors.hexString(s)))` — `ChatColors.hexString` bridges `&`/`§` codes to `<#hex>` (idempotent on already-native strings), MiniMessage applies the custom palette, and `LegacyComponentSerializer(hexColors)` re-emits a `§x`-hex string so item meta stays String-based and SF4's lore/name string read-back keeps working. Then `SlimefunItems.java`'s literals are swept `&`-code → `<#hex>`/`<format>` so the bulk is native (visually identical through the idempotent bridge).

**Tech Stack:** Java 25, Gradle + shadow, SuDo `26.2.0` (`MiniMessages`), Adventure (`LegacyComponentSerializer`, MiniMessage), the vendored `ChatColors` bridge, MockBukkit + JUnit 5.

## Global Constraints

- SuDo `dev.yanianz:sudo-api:26.2.0` (already the dep). No SuDo change. `ChatColors` stays (SF2d removes it).
- Custom palette = ChatColors' `HEX_TAG_MAP`: `&0`→`#2B2D42`, `&1`→`#1D3557`, `&2`→`#2A9D8F`, `&3`→`#457B9D`, `&4`→`#BC4742`, `&5`→`#9B5DE5`, `&6`→`#E9C46A`, `&7`→`#8D99AE`, `&8`→`#4A4E69`, `&9`→`#219EBC`, `&a`→`#2A9D8F`, `&b`→`#8ECAE6`, `&c`→`#E76F51`, `&d`→`#CDB4DB`, `&e`→`#F4A261`, `&f`→`#F1FAEE`; `&l`→`bold`, `&m`→`strikethrough`, `&n`→`underline`, `&o`→`italic`, `&k`→`obfuscated`, `&r`→`reset`.
- Item meta MUST stay String-based (`setDisplayName(String)`/`setLore(List<String>)`) — never Adventure `Component` setters — so lore/name read-back (`ChatColor.stripColor`, `contains("<Type>")`, `replace("<ID>", …)`) keeps working.
- Scope: only `SlimefunItemStack.java` (render) + `SlimefunItems.java` (literals). Other files' literals = SF2c; `ChatColors` deletion = SF2d.
- Every commit ends with: `Claude-Session: https://claude.ai/code/session_013YFwH85eQ1mJXua7Hqcu1k`
- Commit by explicit paths. Trust `./gradlew`, not editor/LSP.

---

### Task 1: `SlimefunItemStack` render bridge (String-preserving) + read-back unit test

**Files:**
- Modify: `src/main/java/io/github/thebusybiscuit/slimefun4/api/items/SlimefunItemStack.java`
- Test: `src/test/java/io/github/thebusybiscuit/slimefun4/api/items/TestSlimefunItemStackRender.java` (new)

**Interfaces:**
- Consumes: `dev.yanianz.sudo.common.MiniMessages.component(String)`; `ChatColors.hexString(String)`; `LegacyComponentSerializer`.
- Produces: `SlimefunItemStack` items whose name/lore are `§x`-hex strings rendering the custom palette; a private `static String render(String)` helper (package-visible for the test via a package-private static, see Step 1).

- [ ] **Step 1: Write the failing test**

`SlimefunItemStack.render(...)` will be `private`; to unit-test it, make it **package-private static** (`static String render(String s)`). Create the test in the same package:

```java
package io.github.thebusybiscuit.slimefun4.api.items;

import org.bukkit.ChatColor;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class TestSlimefunItemStackRender {

    @Test
    void placeholderSurvivesAndStripColorYieldsPlainText() {
        String rendered = SlimefunItemStack.render("&7Type: &b<Type>");
        // The <Type> runtime placeholder must survive rendering as a literal substring.
        assertTrue(rendered.contains("<Type>"), "placeholder <Type> must survive render");
        // Reading the lore back and stripping color must yield the plain text SF4 matches on.
        assertEquals("Type: <Type>", ChatColor.stripColor(rendered));
    }

    @Test
    void appliesHexPaletteAndIsIdempotentOnNativeInput() {
        // Legacy &7 renders to the custom palette hex (#8D99AE) as a §x-hex string.
        String fromLegacy = SlimefunItemStack.render("&7Grey");
        String fromNative = SlimefunItemStack.render("<#8D99AE>Grey");
        // Both inputs render identically (bridge is idempotent on native <#hex>).
        assertEquals(fromLegacy, fromNative);
        assertEquals("Grey", ChatColor.stripColor(fromLegacy));
    }
}
```

- [ ] **Step 2: Run the test — expect FAIL**

Run: `./gradlew test --tests "io.github.thebusybiscuit.slimefun4.api.items.TestSlimefunItemStackRender"`
Expected: FAIL — `render` does not exist (compile error).

- [ ] **Step 3: Add the `render` helper + the serializer field**

In `SlimefunItemStack.java`, add imports:
```java
import dev.yanianz.sudo.common.MiniMessages;
import io.github.thebusybiscuit.slimefun4.libraries.dough.common.ChatColors;
import net.kyori.adventure.text.serializer.legacy.LegacyComponentSerializer;
```
Add a static field + helper (near the top of the class body):
```java
    private static final LegacyComponentSerializer LEGACY =
        LegacyComponentSerializer.builder().hexColors().build();

    /**
     * Renders a legacy {@code &}-coded or native MiniMessage string to a §x-hex legacy string
     * using SuDo MiniMessage + the custom {@link ChatColors} palette. Kept String-based so item
     * name/lore read-back (ChatColor.stripColor, contains("<Type>"), replace("<ID>", …)) works.
     */
    static String render(String s) {
        return LEGACY.serialize(MiniMessages.component(ChatColors.hexString(s)));
    }
```

- [ ] **Step 4: Replace the ~8 render sites**

Change every `ChatColor.translateAlternateColorCodes('&', name)` → `render(name)` and every
`ChatColor.translateAlternateColorCodes('&', line)` → `render(line)` in the constructors (the
display-name sites and the per-lore-line sites). Keep `setDisplayName(String)` / `setLore(List<String>)`
exactly as they are (String-based). Do NOT switch to `displayName(Component)`/`lore(List<Component>)`.

Run `grep -n "translateAlternateColorCodes" src/main/java/io/github/thebusybiscuit/slimefun4/api/items/SlimefunItemStack.java` → expect no output after this step. If `org.bukkit.ChatColor` is now unused in the file, remove its import (grep the file for `ChatColor` first; it may still be used elsewhere — leave the import if so).

- [ ] **Step 5: Run the test — expect PASS**

Run: `./gradlew test --tests "io.github.thebusybiscuit.slimefun4.api.items.TestSlimefunItemStackRender"`
Expected: PASS (2 tests).

**If `placeholderSurvivesAndStripColorYieldsPlainText` fails because MiniMessage mangled `<Type>`:**
Adventure MiniMessage leaves unresolved tags as literal text by default, so `<Type>` should survive.
If it does not (the assertion shows `<Type>` gone or an exception), the fix is to make `render`
pre-escape non-palette angle brackets before MiniMessage and unescape after — but first confirm the
actual behavior from the failure; do not pre-emptively add escaping. Report the exact failure if it
occurs.

- [ ] **Step 6: Full build**

Run: `./gradlew build`
Expected: BUILD SUCCESSFUL; all existing tests pass (item creation across the codebase now routes
through `render`).

- [ ] **Step 7: Commit**

```bash
git add src/main/java/io/github/thebusybiscuit/slimefun4/api/items/SlimefunItemStack.java src/test/java/io/github/thebusybiscuit/slimefun4/api/items/TestSlimefunItemStackRender.java
git commit -m "$(cat <<'MSG'
feat(items): render SlimefunItemStack name/lore via custom hex palette (§x-hex, read-back safe)

Route name/lore through MiniMessages.component(ChatColors.hexString(...)) serialized back to a
§x-hex legacy string. Applies the custom hex palette to all SlimefunItemStack items while keeping
String-based meta so lore/name read-back (stripColor, <Type>/<ID> placeholders) keeps working.

Claude-Session: https://claude.ai/code/session_013YFwH85eQ1mJXua7Hqcu1k
MSG
)"
```

---

### Task 2: Convert `SlimefunItems.java`'s 1086 legacy literals to native MiniMessage

**Files:**
- Modify: `src/main/java/io/github/thebusybiscuit/slimefun4/implementation/SlimefunItems.java`

**Interfaces:**
- Consumes: the Task 1 render bridge (renders both `&` and `<#hex>` identically). No new symbols.
- Produces: `SlimefunItems.java` with native MiniMessage literals; item appearance unchanged (the bridge already applied the palette in Task 1).

**Preconditions (verified during planning):** the file's `&` characters are only legacy codes inside string literals, except two `& ` (ampersand-space) prose cases (`TAPE_MEASURE`, `MULTI_TOOL_LORE`) which are not codes; no `&x…` hex-expansion form; only these codes occur: `&0-9`, `&a-f`, `&l`, `&o`. The runtime placeholders `<Type>`/`<ID>` are not codes and must be left intact.

- [ ] **Step 1: Snapshot representative lines (before)**

Run:
```bash
sed -n '38p;42p;71p;779p' src/main/java/io/github/thebusybiscuit/slimefun4/implementation/SlimefunItems.java
```
Record these four lines — you will diff them after the sweep to confirm correctness (line 38 has consecutive codes `&a&o`; 42 has a `<Type>` placeholder; 71 is `BACKPACK_ID = "&7ID: <ID>"`; 779 is a normal item).

- [ ] **Step 2: Apply the code→tag sweep**

Run this perl (single pass; each `&code` is a distinct token, order-independent; `& ` prose and
`<Type>`/`<ID>` are untouched because they are not `&`+code sequences):

```bash
perl -pi -e '
  s/&0/<#2B2D42>/g; s/&1/<#1D3557>/g; s/&2/<#2A9D8F>/g; s/&3/<#457B9D>/g;
  s/&4/<#BC4742>/g; s/&5/<#9B5DE5>/g; s/&6/<#E9C46A>/g; s/&7/<#8D99AE>/g;
  s/&8/<#4A4E69>/g; s/&9/<#219EBC>/g; s/&a/<#2A9D8F>/g; s/&b/<#8ECAE6>/g;
  s/&c/<#E76F51>/g; s/&d/<#CDB4DB>/g; s/&e/<#F4A261>/g; s/&f/<#F1FAEE>/g;
  s/&l/<bold>/g; s/&m/<strikethrough>/g; s/&n/<underline>/g; s/&o/<italic>/g;
  s/&k/<obfuscated>/g; s/&r/<reset>/g;
' src/main/java/io/github/thebusybiscuit/slimefun4/implementation/SlimefunItems.java
```

- [ ] **Step 3: Verify the sweep**

Run:
```bash
grep -nE '&[0-9a-fk-orA-FK-OR]' src/main/java/io/github/thebusybiscuit/slimefun4/implementation/SlimefunItems.java
```
Expected: **no output** — no legacy `&`-code remains (the only `&` left are the two `& ` prose cases). Confirm those two survive:
```bash
grep -nE '& ' src/main/java/io/github/thebusybiscuit/slimefun4/implementation/SlimefunItems.java
```
Expected: the `TAPE_MEASURE` + `MULTI_TOOL_LORE` lines, with their `& ` intact.

Then re-print the four snapshot lines and confirm they converted correctly:
```bash
sed -n '38p;42p;71p;779p' src/main/java/io/github/thebusybiscuit/slimefun4/implementation/SlimefunItems.java
```
Expected e.g. line 779: `"<#8D99AE>Cargo Node <#E76F51>(Connector)", "", "<#F1FAEE>Cargo Connector Pipe"`; line 42's `<Type>` placeholder still present as `<#8ECAE6><Type>`; line 71: `"<#8D99AE>ID: <ID>"`.

- [ ] **Step 4: Full build + tests**

Run: `./gradlew build`
Expected: BUILD SUCCESSFUL; all tests pass. In particular the Task 1 render test and any spawner/
backpack tests still pass (the `<Type>`/`<ID>` placeholders survive rendering and read-back).

If a converted literal breaks compilation (e.g. an unintended replacement inside a non-display
string), inspect the offending line, revert that specific substitution, and handle it manually.
If the build fails at runtime-only paths (not caught by compile), rely on the existing tests; if a
spawner/backpack test fails on placeholder handling, that is a real regression — STOP and report
with the failing test.

- [ ] **Step 5: Commit**

```bash
git add src/main/java/io/github/thebusybiscuit/slimefun4/implementation/SlimefunItems.java
git commit -m "$(cat <<'MSG'
refactor(items): convert SlimefunItems 1086 legacy &-literals to native MiniMessage hex

Sweep &-codes -> <#hex>/<format> per the custom ChatColors palette. Visually identical (the
render bridge already applied the palette); makes the item-registry literals native MiniMessage.
<Type>/<ID> runtime placeholders and & -in-prose left intact.

Claude-Session: https://claude.ai/code/session_013YFwH85eQ1mJXua7Hqcu1k
MSG
)"
```

---

## Self-Review

**Spec coverage:**
- Render bridge (String-preserving §x-hex, read-back safe) → Task 1. Custom palette applied → Task 1 render + Task 2 map. Placeholder/read-back safety → Task 1 test + Task 2 preservation.
- Convert SlimefunItems 1086 literals → Task 2 sweep + verify. Escaping/precondition handling → Task 2 preconditions + Step 4 contingency.
- Success criteria (no translateAlternateColorCodes in SlimefunItemStack; no `&`-codes in SlimefunItems; build+tests green) → Task 1 Step 4/6, Task 2 Step 3/4.

**Placeholder scan:** No TBD/TODO. Full render code, full palette perl, exact grep expectations, complete commit messages. The MiniMessage-`<Type>` contingency is a verify-then-act step keyed on an actual test failure, not deferred work.

**Type consistency:** `render(String)→String` (package-private static) defined in Task 1, used by the Task 1 test and by the Task 2 bridge implicitly. `LEGACY` field + `MiniMessages.component` + `ChatColors.hexString` consistent. Palette map identical between Global Constraints and Task 2 perl.

**Scope check:** two tasks — render bridge (with test) then bulk literal sweep — each ending in a green build. `SlimefunItemStack.java` + `SlimefunItems.java` only; other files + ChatColors deletion deferred to SF2c/SF2d.
