# Slimefun4 SF2d — Remove `org.bukkit.ChatColor` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove every `org.bukkit.ChatColor` reference from `src/main` (outside `libraries/dough/`) — constants → inline `<#hex>` tags, `stripColor` → `MiniMessages.plain(MiniMessages.stripLegacy(...))`, `ChatColor`-typed fields/params → `String`.

**Architecture:** Work by shape in dependency order so the build stays compilable: first `stripColor` calls, then the `ChatColor`-typed members (structural), then the bulk color-constant substitutions (safe once no `ChatColor`-typed context remains), then imports + sink/read-back audit. Colors use the custom ChatColors palette (consistent with SF2b/c); no new helper class — inline `<#hex>` literals + SuDo `MiniMessages`.

**Tech Stack:** Java 25, Gradle + shadow, SuDo `26.2.0` (`MiniMessages.plain`/`stripLegacy`/`component`), the vendored `ChatColors` bridge (kept until SF2f), MockBukkit + JUnit 5, `perl`/`sed` for mechanical constant substitution.

## Global Constraints

- SuDo `dev.yanianz:sudo-api:26.2.0`. No SuDo change. `libraries/dough` (ChatColors, CustomItemStack) untouched (SF2e/f).
- Palette (ChatColors `CHATCOLOR_HEX_MAP`): `BLACK`→`<#2B2D42>`, `DARK_BLUE`→`<#1D3557>`, `DARK_GREEN`→`<#2A9D8F>`, `DARK_AQUA`→`<#457B9D>`, `DARK_RED`→`<#BC4742>`, `DARK_PURPLE`→`<#9B5DE5>`, `GOLD`→`<#E9C46A>`, `GRAY`→`<#8D99AE>`, `DARK_GRAY`→`<#4A4E69>`, `BLUE`→`<#219EBC>`, `GREEN`→`<#2A9D8F>`, `AQUA`→`<#8ECAE6>`, `RED`→`<#E76F51>`, `LIGHT_PURPLE`→`<#CDB4DB>`, `YELLOW`→`<#F4A261>`, `WHITE`→`<#F1FAEE>`; `RESET`→`<reset>`.
- `stripColor(s)` → `MiniMessages.plain(MiniMessages.stripLegacy(s))` (import `dev.yanianz.sudo.common.MiniMessages`).
- Every commit ends with: `Claude-Session: https://claude.ai/code/session_013YFwH85eQ1mJXua7Hqcu1k`
- Commit by explicit paths. Trust `./gradlew`, not editor/LSP.

---

### Task 1: Replace `ChatColor.stripColor` with `MiniMessages.plain(stripLegacy(...))`

**Files (18 code sites):** `core/services/profiler/PerformanceSummary.java`, `utils/ChatUtils.java`, `implementation/guide/SurvivalSlimefunGuide.java` (×2), `implementation/items/LimitedUseItem.java`, `implementation/items/blocks/AbstractMonsterSpawner.java` (×2), `implementation/items/magical/KnowledgeTome.java`, `implementation/items/magical/talismans/Talisman.java`, `implementation/items/electric/gadgets/MultiTool.java` (×3), `api/items/ItemGroup.java`, `api/recipes/RecipeType.java`, `api/gps/GPSNetwork.java`, `api/researches/Research.java`. (The `SlimefunItemStack.java:68` hit is a javadoc comment — update its text only.)

**Interfaces:**
- Consumes: `MiniMessages.plain(String)`, `MiniMessages.stripLegacy(String)` (SuDo 26.2.0).
- Produces: no `ChatColor.stripColor` in `src/main`.

- [ ] **Step 1: Replace every `ChatColor.stripColor(` call**

In each file, add `import dev.yanianz.sudo.common.MiniMessages;` (if absent) and replace
`ChatColor.stripColor(X)` with `MiniMessages.plain(MiniMessages.stripLegacy(X))`. Mechanical:
```bash
grep -rl "ChatColor.stripColor(" --include='*.java' src/main | grep -v '/libraries/dough/' | while IFS= read -r f; do
  perl -0777 -pi -e 's/ChatColor\.stripColor\(/MiniMessages.plain(MiniMessages.stripLegacy(/g' "$f"
done
```
This adds ONE `(` — the matching close paren must be added. Because the original `stripColor(X)`
already has its close `)`, the transform produces `MiniMessages.plain(MiniMessages.stripLegacy(X)`
which is missing one `)`. **Do NOT use the blind perl above.** Instead, per site, hand-edit each
`ChatColor.stripColor(X)` → `MiniMessages.plain(MiniMessages.stripLegacy(X))` (add the extra
closing paren). There are 18 sites — do them individually (the argument `X` varies:
`line`, `itemName`, `input.toLowerCase(...)`, `item.getItemMeta().getLore().get(1)`, etc.). After
each, the expression is balanced.

- [ ] **Step 2: Fix the `SlimefunItemStack.java:68` javadoc mention**

It reads `... (ChatColor.stripColor, contains("&lt;Type&gt;"), ...)`. Change `ChatColor.stripColor`
to `MiniMessages.plain` in that comment for accuracy.

- [ ] **Step 3: Add the `MiniMessages` import where missing + verify**

Run: `grep -rl "MiniMessages.plain(MiniMessages.stripLegacy" --include='*.java' src/main | while read f; do grep -q "import dev.yanianz.sudo.common.MiniMessages;" "$f" || echo "MISSING IMPORT: $f"; done`
Add the import to any file listed.
Run: `grep -rn "ChatColor.stripColor" --include='*.java' src/main | grep -v '/libraries/dough/'` → no output.

- [ ] **Step 4: Compile**

Run: `./gradlew compileJava`
Expected: BUILD SUCCESSFUL. A missing-paren mistake fails here — fix the offending line.

- [ ] **Step 5: Commit**

```bash
git add -u src/main
git commit -m "$(cat <<'MSG'
refactor(text): replace ChatColor.stripColor with MiniMessages.plain(stripLegacy(...))

§x-hex-aware plain: stripLegacy removes §/§x codes, plain removes any <#hex> tags. Handles both
item lore (§x-hex) and MiniMessage strings. First step of removing org.bukkit.ChatColor.

Claude-Session: https://claude.ai/code/session_013YFwH85eQ1mJXua7Hqcu1k
MSG
)"
```

---

### Task 2: Refactor `ChatColor`-typed fields/params/locals to `String` hex tags

**Files:**
- `core/attributes/Radioactivity.java` (enum field `ChatColor color` + ctor param → `String`)
- `core/services/profiler/PerformanceRating.java` (enum field `ChatColor color` + ctor + `getColor()` → `String`)
- `core/commands/subcommands/VersionsCommand.java` (locals `ChatColor primaryColor/secondaryColor` → `String`)
- `utils/ChatUtils.java` (`crop(ChatColor color, String)` → `crop(String color, String)`)
- `implementation/guide/SurvivalSlimefunGuide.java:342` (caller: `ChatUtils.crop(ChatColor.WHITE, input)` → `crop("<#F1FAEE>", input)`)

**Interfaces:**
- Consumes: the palette (hex tags). `MiniMessages` (already imported where needed).
- Produces: no `ChatColor`-typed members; `getColor()`/`color` are `String` hex tags.

- [ ] **Step 1: `Radioactivity`**

Change `private final ChatColor color;` → `private final String color;`, the constructor param
`ChatColor color` → `String color`, and each enum constant's color argument from `ChatColor.<NAME>`
to the `<#hex>` tag (per the palette). Read the enum constants first (`sed -n '1,50p'`) to map each.
Update any `getColor()`/`.color` consumer to treat it as a `String` (grep
`Radioactivity` usages; if a consumer concatenated `color + "text"`, it still works with the hex
string, now reaching a MiniMessage sink — verify per §Task 4 audit).

- [ ] **Step 2: `PerformanceRating`**

Same shape: field + ctor param + `public ChatColor getColor()` → `public String getColor()`;
enum-constant color args → `<#hex>`. Update consumers of `getColor()` (grep
`PerformanceRating` / `.getColor()` in profiler code) to `String`.

- [ ] **Step 3: `VersionsCommand`**

Change the `ChatColor primaryColor;` / `ChatColor secondaryColor;` locals → `String`, and the
branch assignments (`primaryColor = ChatColor.X;`) → `= "<#hex>";`. Their concatenation use
(`primaryColor + "..."`) then produces a `<#hex>` string; verify it reaches a MiniMessage sink
(§Task 4).

- [ ] **Step 4: `ChatUtils.crop` + caller**

`crop(@Nonnull ChatColor color, @Nonnull String string)` → `crop(@Nonnull String color, @Nonnull String string)`.
Inside, the two `stripColor` calls were already converted in Task 1; the `color + …`
concatenation now uses the hex-tag string. Update the sole caller
`SurvivalSlimefunGuide.java:342` `ChatUtils.crop(ChatColor.WHITE, input)` → `ChatUtils.crop("<#F1FAEE>", input)`.

- [ ] **Step 5: Compile**

Run: `./gradlew compileJava`
Expected: BUILD SUCCESSFUL. Any missed consumer (a `ChatColor` still assigned from the now-`String`
`getColor()`) fails here — fix it.

- [ ] **Step 6: Commit**

```bash
git add -u src/main
git commit -m "$(cat <<'MSG'
refactor(text): ChatColor-typed fields/params -> String hex tags

Radioactivity, PerformanceRating, VersionsCommand, ChatUtils.crop and its caller no longer use the
org.bukkit.ChatColor type; colors are custom-palette <#hex> tag strings.

Claude-Session: https://claude.ai/code/session_013YFwH85eQ1mJXua7Hqcu1k
MSG
)"
```

---

### Task 3: Replace color constants with inline `<#hex>` + remove imports

**Files:** all remaining `src/main` files referencing `ChatColor.<CONSTANT>` / `ChatColor.RESET` /
`ChatColor.COLOR_CHAR` / `ChatColor.translateAlternateColorCodes` (enumerate via grep in Step 1;
`libraries/dough/` excluded).

**Interfaces:**
- Consumes: the palette. `MiniMessages.component` (for the 1 translateAlternate + any raw-sink wrap).
- Produces: zero `org.bukkit.ChatColor` refs/imports in `src/main` outside `libraries/dough/`.

- [ ] **Step 1: Enumerate remaining constant sites**

Run:
```bash
grep -rn "ChatColor\." --include='*.java' src/main | grep -v '/libraries/dough/'
```
After Tasks 1–2, the remaining hits are colour constants, `RESET`, `COLOR_CHAR`,
`translateAlternateColorCodes`. Record the file list.

- [ ] **Step 2: Substitute the 16 colour constants + RESET**

For each remaining file, replace each `ChatColor.<NAME>` with its `<#hex>` string literal. Because
these are now all VALUE uses (types were removed in Task 2), the replacement `ChatColor.GREEN` →
`"<#2A9D8F>"` is valid wherever it appears (concatenation, method arg). Apply per-file:
```bash
grep -rl "ChatColor\." --include='*.java' src/main | grep -v '/libraries/dough/' | while IFS= read -r f; do
perl -0777 -pi -e '
  s/ChatColor\.BLACK\b/"<#2B2D42>"/g;       s/ChatColor\.DARK_BLUE\b/"<#1D3557>"/g;
  s/ChatColor\.DARK_GREEN\b/"<#2A9D8F>"/g;  s/ChatColor\.DARK_AQUA\b/"<#457B9D>"/g;
  s/ChatColor\.DARK_RED\b/"<#BC4742>"/g;    s/ChatColor\.DARK_PURPLE\b/"<#9B5DE5>"/g;
  s/ChatColor\.GOLD\b/"<#E9C46A>"/g;        s/ChatColor\.GRAY\b/"<#8D99AE>"/g;
  s/ChatColor\.DARK_GRAY\b/"<#4A4E69>"/g;   s/ChatColor\.BLUE\b/"<#219EBC>"/g;
  s/ChatColor\.GREEN\b/"<#2A9D8F>"/g;       s/ChatColor\.AQUA\b/"<#8ECAE6>"/g;
  s/ChatColor\.RED\b/"<#E76F51>"/g;         s/ChatColor\.LIGHT_PURPLE\b/"<#CDB4DB>"/g;
  s/ChatColor\.YELLOW\b/"<#F4A261>"/g;      s/ChatColor\.WHITE\b/"<#F1FAEE>"/g;
  s/ChatColor\.RESET\b/"<reset>"/g;
' "$f"
done
```
Order the DARK_* substitutions before the plain ones (done above) so `ChatColor.DARK_GREEN` is not
partially matched by `ChatColor.GREEN` — the `\b` word boundary already prevents this, but the
ordering is belt-and-suspenders.

- [ ] **Step 3: Handle the 3 remaining specials**

- `ChatColor.translateAlternateColorCodes('&', X)` (1 site — grep it) → `MiniMessages.component(X)`
  IF the result goes to a Component sink, else keep the assembled string and wrap at the sink. Grep
  the site, read its sink, apply the SF2c pattern (route through `MiniMessages.component`).
- `ChatColor.COLOR_CHAR` (1 site — grep it) → the literal `'§'` (§). Read the site; if it
  builds a legacy string for a raw sink, route that through `MiniMessages` instead.
- Any `ChatColor.<NAME>.toString()` / `.getChar()` (grep) → the `<#hex>` string already replaces
  the constant; drop the `.toString()` (the value is already a String) — hand-fix.

- [ ] **Step 4: Remove `import org.bukkit.ChatColor;` everywhere it is now unused**

```bash
grep -rl "import org.bukkit.ChatColor;" --include='*.java' src/main | grep -v '/libraries/dough/' | while IFS= read -r f; do
  grep -q "ChatColor\." "$f" || perl -0777 -pi -e 's/^import org\.bukkit\.ChatColor;\n//mg' "$f"
done
grep -rn "org.bukkit.ChatColor\|ChatColor\." --include='*.java' src/main | grep -v '/libraries/dough/'
```
Expected: the final grep returns no output. If a file still references `ChatColor.` (a special from
Step 3 not yet handled), fix it, then re-run.

- [ ] **Step 5: Compile**

Run: `./gradlew compileJava compileTestJava`
Expected: BUILD SUCCESSFUL.

- [ ] **Step 6: Commit**

```bash
git add -u src/main
git commit -m "$(cat <<'MSG'
refactor(text): replace org.bukkit.ChatColor constants with inline <#hex>; remove imports

All colour constants -> custom-palette <#hex> tag literals; RESET -> <reset>; translateAlternate ->
MiniMessages.component; COLOR_CHAR -> §. org.bukkit.ChatColor is now unreferenced in src/main
(outside libraries/dough). Sink/read-back audit follows.

Claude-Session: https://claude.ai/code/session_013YFwH85eQ1mJXua7Hqcu1k
MSG
)"
```

---

### Task 4: Sink + read-back audit, full build & tests

**Files:** any needing a sink wrap or read-back normalization (found in Steps 1–2).

**Interfaces:**
- Consumes: Tasks 1–3. Produces: a green, audited build.

- [ ] **Step 1: Sink audit — `<#hex>` reaching a raw legacy sink**

```bash
grep -rnE '\.sendMessage\("[^"]*<#|\.sendMessage\([^)]*"<#|(setDisplayName|setLore|setLabel)\([^)]*<#|translateAlternateColorCodes\([^)]*<#' --include='*.java' src/main | grep -v '/libraries/dough/'
```
For each hit where the `<#hex>` string is passed to a String/legacy overload (not through
`CustomItemStack`/`ChatColors`/`SlimefunItemStack.render`/localization `MiniMessages`), wrap the
assembled argument in `MiniMessages.component(...)`. (The SF2c `CargoManager` fix is the template.)
Also grep for a `<#hex>` string built from a former constant then `sender.sendMessage(that)` — trace
locals from `VersionsCommand`/message code.

- [ ] **Step 2: Read-back audit — comparisons vs a former-constant string**

```bash
grep -rnE '(getLore|getDisplayName)\(\)[^;]*(equals|startsWith|contains|matcher)' --include='*.java' src/main | grep -v '/libraries/dough/' | grep -vE 'hexString|stripLegacy|MiniMessages.plain'
```
For each, if it compares item text (now `§x`-hex) against a `"<#hex>…"` string that came from a
former `ChatColor.X + "…"` constant, make it format-robust: compare
`MiniMessages.plain(MiniMessages.stripLegacy(line))` to the plain text, or normalize both sides via
`ChatColors.hexString(...)`. Leave self-consistent set-and-read pairs (both sides use the same new
constant) as-is.

- [ ] **Step 3: Full build + tests**

Run: `./gradlew build`
Expected: BUILD SUCCESSFUL; all tests pass. Update any test asserting old `ChatColor`/`§`-format
text to compare `MiniMessages.plain(MiniMessages.stripLegacy(...))` / stripped content (assert
content, not colour bytes). A failing lore/name test is either a real read-back regression (Step 2)
or a stale colour-format assertion (fix the test).

- [ ] **Step 4: Final ChatColor audit**

Run: `grep -rn "org.bukkit.ChatColor" --include='*.java' src/main src/test | grep -v '/libraries/dough/'`
Expected: no output (zero `org.bukkit.ChatColor` anywhere in src/main + src/test outside dough).

- [ ] **Step 5: Spot-check per shape**

Confirm by inspection: a converted `"<#hex>" + msg` message renders coloured (reaches a MiniMessage
sink); a `plain(stripLegacy(...))` read yields the expected plain text; `Radioactivity`/
`PerformanceRating`/`VersionsCommand` colour usages render; `ChatUtils.crop("<#F1FAEE>", input)` works.

- [ ] **Step 6: Commit (if fixes were needed)**

```bash
git add -u src/main src/test
git commit -m "$(cat <<'MSG'
fix(text): sink-route + read-back-normalize after ChatColor removal

Wrap <#hex> strings reaching raw legacy sinks in MiniMessages.component; normalize lore/name
comparisons against former-constant strings; update tests asserting old colour-format text.

Claude-Session: https://claude.ai/code/session_013YFwH85eQ1mJXua7Hqcu1k
MSG
)"
```
(If Steps 1–2 found nothing to fix and all tests passed, no commit — Task 3 already left the build green.)

---

## Self-Review

**Spec coverage:**
- Zero ChatColor + import removal → Tasks 3–4 (audit). Constants → hex → Task 3. `stripColor` →
  `plain(stripLegacy)` → Task 1. Type-usages → String → Task 2. `RESET`/`COLOR_CHAR`/
  `translateAlternate` → Task 3 Step 3. Sink-route + read-back → Task 4. Build/tests → Tasks 1/2/3
  compile + Task 4 full.

**Placeholder scan:** No TBD/TODO. Palette map given in Global Constraints + Task 3 perl; the
`stripColor` extra-paren pitfall is called out with the hand-edit instruction (not a blind sweep);
special-case handling (translateAlternate/COLOR_CHAR/toString) is concrete grep-then-apply.

**Type consistency:** `MiniMessages.plain`/`stripLegacy`/`component` used identically across tasks.
`getColor()`→`String` (Task 2) consumed as String (Task 3 audit). Palette tags identical between
Global Constraints, Task 2 (enum args), and Task 3 (perl). `crop(String, String)` signature matches
its updated caller.

**Scope check:** four tasks by shape, each ending in a compile (1–3) or full green build (4).
`CustomItemStack`/`ChatColors`/dough untouched (SF2e/f). The dominant risk (sink-routing +
read-back at 188-ref scale) is isolated to Task 4's audit with concrete greps + the SF2c template.
