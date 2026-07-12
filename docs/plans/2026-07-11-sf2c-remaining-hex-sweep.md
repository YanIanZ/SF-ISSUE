# Slimefun4 SF2c — Remaining `&`-Literal Hex Sweep Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Convert the remaining 466 legacy `&`-code string literals (57 files, outside `SlimefunItems.java` and `libraries/dough/`) to native `<#hex>`/`<format>` MiniMessage, keeping rendering correct and lore/name read-back robust.

**Architecture:** A **literal-scoped** converter applies ChatColors' palette (`&7`→`<#8D99AE>`, `&l`→`<bold>`, etc.) only inside Java string literals, so bitwise/logical `&` in code is never touched. The sinks these literals feed already bridge `&`/`<#hex>` to MiniMessage (localization `MiniMessages.component`, `CustomItemStack`/`ChatColors.hexString`, `SlimefunItemStack.render`), so the conversion is visually neutral. A read-back audit makes any lore/name comparisons in touched files format-robust (the SF2b landmine).

**Tech Stack:** Java 25, Gradle + shadow, SuDo `26.2.0`, the vendored `ChatColors` bridge, MockBukkit + JUnit 5, `perl` for the literal-scoped sweep.

## Global Constraints

- SuDo `dev.yanianz:sudo-api:26.2.0` (already the dep). No SuDo change.
- Palette (ChatColors `HEX_TAG_MAP`): `&0`→`<#2B2D42>`, `&1`→`<#1D3557>`, `&2`→`<#2A9D8F>`, `&3`→`<#457B9D>`, `&4`→`<#BC4742>`, `&5`→`<#9B5DE5>`, `&6`→`<#E9C46A>`, `&7`→`<#8D99AE>`, `&8`→`<#4A4E69>`, `&9`→`<#219EBC>`, `&a`→`<#2A9D8F>`, `&b`→`<#8ECAE6>`, `&c`→`<#E76F51>`, `&d`→`<#CDB4DB>`, `&e`→`<#F4A261>`, `&f`→`<#F1FAEE>`; `&l`→`<bold>`, `&m`→`<strikethrough>`, `&n`→`<underline>`, `&o`→`<italic>`, `&k`→`<obfuscated>`, `&r`→`<reset>`.
- Convert `&`-codes **only inside double-quoted string literals** (never bare Java `&`). All codes in these files are lowercase (verified — zero uppercase codes, no `R&D`-style word-ampersand false positives; adjacencies are consecutive codes).
- Scope: `src/main` `.java` files EXCEPT `SlimefunItems.java` (done in SF2b) and anything under `libraries/dough/`. Do NOT touch `org.bukkit.ChatColor` API usage (SF2d), `CustomItemStack` (SF2e), or `ChatColors`/dough (SF2f).
- Every commit ends with: `Claude-Session: https://claude.ai/code/session_013YFwH85eQ1mJXua7Hqcu1k`
- Commit by explicit paths. Trust `./gradlew`, not editor/LSP.

---

### Task 1: Literal-scoped hex sweep of the 57 files

**Files:**
- Modify: every `src/main` `.java` file containing a legacy `&`-code string literal EXCEPT
  `implementation/SlimefunItems.java` and anything under `libraries/dough/` (57 files; enumerate via
  the grep in Step 1).

**Interfaces:**
- Consumes: nothing (mechanical text transform). The sinks (`SlimefunItemStack.render`,
  `CustomItemStack`/`ChatColors.hexString`, localization `MiniMessages.component`) already bridge.
- Produces: those files with native MiniMessage literals; no legacy `&`-codes in literals.

- [ ] **Step 1: Enumerate the target files + snapshot samples**

Run:
```bash
FILES=$(grep -rlE '"[^"]*&[0-9a-fk-or]' --include='*.java' src/main | grep -vE '/libraries/dough/|SlimefunItems.java')
echo "$FILES" | wc -l          # expect ~57
echo "$FILES" | sed 's#src/main/java/io/github/thebusybiscuit/slimefun4/##' | sort
```
Record the file list. Snapshot 3 representative lines to diff after:
```bash
grep -rnE '"[^"]*&[0-9a-fk-or]' $FILES | head -3
```

- [ ] **Step 2: Precondition check — no uppercase codes / no bitwise-in-literal surprises**

Run:
```bash
echo "$FILES" | xargs grep -rhoE '&[A-FK-OR]' | sort -u    # expect EMPTY (no uppercase codes)
echo "$FILES" | xargs grep -rhoE '&&' | head              # logical-AND: must be OUTSIDE literals (converter is literal-scoped, so untouched)
```
Expected: no uppercase `&`-codes. (Logical/bitwise `&` in code is fine — the converter only edits inside string literals.)

- [ ] **Step 3: Apply the literal-scoped converter to each file**

For each file in `$FILES`, run this perl (it matches each Java string literal `"…"`, handling
escaped quotes, and applies the palette map **only inside** the literal — Java `&`/`&&`/bitwise
outside literals is never touched):

```bash
for f in $FILES; do
perl -0777 -pi -e '
s{"((?:\\.|[^"\\])*)"}{
  my $l = $1;
  $l =~ s/&0/<#2B2D42>/g; $l =~ s/&1/<#1D3557>/g; $l =~ s/&2/<#2A9D8F>/g; $l =~ s/&3/<#457B9D>/g;
  $l =~ s/&4/<#BC4742>/g; $l =~ s/&5/<#9B5DE5>/g; $l =~ s/&6/<#E9C46A>/g; $l =~ s/&7/<#8D99AE>/g;
  $l =~ s/&8/<#4A4E69>/g; $l =~ s/&9/<#219EBC>/g; $l =~ s/&a/<#2A9D8F>/g; $l =~ s/&b/<#8ECAE6>/g;
  $l =~ s/&c/<#E76F51>/g; $l =~ s/&d/<#CDB4DB>/g; $l =~ s/&e/<#F4A261>/g; $l =~ s/&f/<#F1FAEE>/g;
  $l =~ s/&l/<bold>/g; $l =~ s/&m/<strikethrough>/g; $l =~ s/&n/<underline>/g; $l =~ s/&o/<italic>/g;
  $l =~ s/&k/<obfuscated>/g; $l =~ s/&r/<reset>/g;
  "\"" . $l . "\""
}ge;
' "$f"
done
```

- [ ] **Step 4: Verify the sweep**

Run:
```bash
echo "$FILES" | xargs grep -nE '"[^"]*&[0-9a-fk-or]' | grep -vE '& [A-Za-z]|&amp;'    # legacy codes left in literals
```
Expected: **no output** — no `&`-code remains inside a string literal (the `& `-in-prose cases,
e.g. `"Crouch & Right Click"`, are excluded by the `& [A-Za-z]` filter and are correct to keep).

Re-print the Step 1 snapshot lines and confirm they converted correctly (codes → `<#hex>`/`<format>`,
placeholders like `<Type>`/`<ID>`/`%usage%` and `& `-prose intact).

- [ ] **Step 5: Sink re-verification — no hex literal at a raw legacy sink**

Run:
```bash
grep -rnE '\.sendMessage\("[^"]*<#' --include='*.java' src/main         # hex literal to String sendMessage
grep -rnE '(setLore|setDisplayName)\("[^"]*<#|setLabel\("[^"]*<#' --include='*.java' src/main
```
Expected: no output. If any appear, the literal reaches a raw legacy sink — wrap it in
`MiniMessages.component(...)` (import `dev.yanianz.sudo.common.MiniMessages`) or route through
`ChatColors.hexString(...)` consistent with the surrounding code. (Survey found 0 such sites; this
step re-confirms after the sweep.)

- [ ] **Step 6: Compile**

Run: `./gradlew compileJava compileTestJava`
Expected: BUILD SUCCESSFUL. A mangled literal (unbalanced quote from an escaped-quote edge case)
would fail here — inspect and hand-fix the offending file, then recompile.

- [ ] **Step 7: Commit**

```bash
git add -u src/main
git commit -m "$(cat <<'MSG'
refactor(items): sweep remaining 466 legacy &-literals to native MiniMessage hex

Literal-scoped conversion (inside string literals only) of &-codes -> <#hex>/<format> per the
custom ChatColors palette across 57 files (item classes, GUI/menu helpers, messages, holograms).
Visually neutral: sinks already bridge &/<#hex> (SlimefunItemStack.render, CustomItemStack,
ChatColors.hexString, localization MiniMessages.component). Placeholders and & -in-prose intact.

Claude-Session: https://claude.ai/code/session_013YFwH85eQ1mJXua7Hqcu1k
MSG
)"
```

---

### Task 2: Read-back audit + full build/tests

**Files:**
- Modify: any file in `src/main` that compares item/GUI lore or display name against a hardcoded
  color constant / `hexString` needle and breaks under the reskin (enumerated in Step 1).

**Interfaces:**
- Consumes: the Task 1 sweep.
- Produces: format-robust read-back; green build with all tests.

- [ ] **Step 1: Find lore/name comparisons that assume a color format**

Run:
```bash
grep -rnE '(getLore|getDisplayName)\(\).*(equals|startsWith|contains|matcher)|ChatColor\.[A-Z_]+ ?\+ ?"' --include='*.java' src/main | grep -vE '/libraries/dough/'
```
Review each hit. A site is a risk when it matches item text (from `SlimefunItemStack`/`CustomItemStack`,
now `§x`-hex / `<#hex>`) against a hardcoded legacy color. Cross-reference the known set from SF2b
(the pattern to apply is: normalize BOTH sides via `ChatColors.hexString(...)`, or compare
`ChatColor.stripColor(...)` text — as done in `AbstractCraftingTable`, `BackpackListener:161`,
`PlayerProfile.getBackpack`, `LimitedUseItem`).

- [ ] **Step 2: Fix each real read-back site**

For each confirmed risk, apply the established normalization. Example shape (already used in SF2b):
```java
// before: if (line.startsWith(ChatColors.hexString("&7Foo"))) { ... line.replace(...) }
String normalized = ChatColors.hexString(line);
if (normalized.startsWith(ChatColors.hexString("&7Foo"))) {
    String rest = normalized.replace(ChatColors.hexString("&7Foo"), "");
    ...
}
// or, when only content matters: ChatColor.stripColor(line).equals("Foo")
```
Do NOT rewrite comparisons that are self-consistent (a value set and read via the same runtime
constant — e.g. soulbound `SOULBOUND_LORE`, `FireworksListener`) or display-only. Leave
`org.bukkit.ChatColor` API usage otherwise intact (SF2d).

- [ ] **Step 3: Update any test asserting old color-format text**

If a test asserts `§`/`ChatColor`-format lore/name that the reskin changed, update it to compare
`ChatColor.stripColor(...)` / normalized text (as `TestBackpackCommand` was updated in SF2b). Keep
the assertion meaningful (assert the content, not the color bytes).

- [ ] **Step 4: Full build + tests**

Run: `./gradlew build`
Expected: BUILD SUCCESSFUL; all tests pass. If a test fails on lore/name text, it is either a real
read-back regression (fix per Step 2) or a stale color-format assertion (fix per Step 3).

- [ ] **Step 5: Spot-check per sink type**

Confirm by inspection that a swept literal reaching each sink renders (no raw `<#…>`):
- a `CustomItemStack` GUI button (`utils/ChestMenuUtils.java`);
- a localized message (`SlimefunLocalization` path);
- an item class (e.g. `implementation/items/cargo/*`);
- a hologram caller (`core/attributes/HologramOwner.java`).
Each should pass its swept literal to a MiniMessage-bridging sink (grep the call site).

- [ ] **Step 6: Commit**

```bash
git add -u src/main src/test
git commit -m "$(cat <<'MSG'
fix(items): make lore/name read-back format-robust after SF2c hex sweep

Normalize lore/name comparisons touched by the reskin (hexString both sides / stripColor content),
and update tests asserting old color-format text. Self-consistent and display-only sites left as-is;
org.bukkit.ChatColor API removal deferred to SF2d.

Claude-Session: https://claude.ai/code/session_013YFwH85eQ1mJXua7Hqcu1k
MSG
)"
```

---

## Self-Review

**Spec coverage:**
- Convert 466 literals (57 files, literal-scoped, palette-exact) → Task 1. Sink re-verification → Task 1 Step 5. Read-back audit + fixes → Task 2. Build/tests + spot-check → Task 2. Carry SF2b deferrals (AbstractEnchantmentMachine, needle unify) → noted in Task 2 Step 2 scope (org.bukkit.ChatColor left for SF2d).

**Placeholder scan:** No TBD/TODO. Exact grep enumerations, the full literal-scoped perl with the complete palette, exact verification greps, complete commit messages. The read-back audit uses the concrete SF2b normalization pattern with example code, not a vague "handle edge cases."

**Type consistency:** Palette map identical between Global Constraints and the Task 1 perl. The normalization pattern (`ChatColors.hexString(...)` both sides / `ChatColor.stripColor(...)`) matches the SF2b-established sites referenced. No new symbols introduced.

**Scope check:** two tasks — the mechanical sweep (ends at a clean compile) then the read-back audit (ends at green build + tests). `org.bukkit.ChatColor` / `CustomItemStack` / `ChatColors`+dough explicitly deferred to SF2d/e/f.
