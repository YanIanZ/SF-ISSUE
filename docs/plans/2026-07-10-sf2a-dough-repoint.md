# Slimefun4 SF2a — dough Repoint Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Repoint all callers of the vendored `dough` `Config` shim to `dev.yanianz.sudo.config.Config` and delete the three redundant dough shim files (`config/Config`, and the dead `skins/PlayerHead` + `skins/CustomGameProfile`).

**Architecture:** The dough `Config` is a pure add-nothing subclass of `dev.yanianz.sudo.config.Config` (five identical `super(...)` constructors, overrides nothing). Callers are repointed by swapping the import (19 files) or the fully-qualified reference (BlockStorage, 3 sites) to the SuDo class. The two dough skins files have zero importers (call sites already use SuDo directly) and are deleted as dead code. Pure mechanical change, no behavior difference — the build + existing tests are the gate.

**Tech Stack:** Java 25, Gradle (Kotlin DSL) + shadow, SuDo `26.2.0`, Paper 26.2 (compileOnly), MockBukkit.

## Global Constraints

- SuDo consumed as `dev.yanianz:sudo-api:26.2.0` (already the dependency). No SuDo change.
- Scope: only `Config` repoint + the three shim deletions. `CustomItemStack` and `ChatColors` stay (SF2b–d).
- Every git commit message ends with: `Claude-Session: https://claude.ai/code/session_013YFwH85eQ1mJXua7Hqcu1k`
- Commit by explicit paths (never `git add -A` — `.gradle/` churn present).
- Trust `./gradlew` output, not editor/LSP diagnostics.

---

### Task 1: Repoint Config to SuDo + delete the three redundant dough shims

**Files:**
- Modify (import swap — 19 files): `core/SlimefunRegistry.java`, `core/attributes/DamageableItem.java`, `core/networks/NetworkManager.java`, `core/services/CustomTextureService.java`, `core/services/PerWorldSettingsService.java`, `core/services/sounds/SoundService.java`, `core/services/PermissionsService.java`, `core/services/UpdaterService.java`, `core/services/localization/SlimefunLocalization.java`, `core/services/github/GitHubService.java`, `storage/backend/legacy/LegacyStorage.java`, `implementation/Slimefun.java`, `implementation/items/androids/Script.java`, `api/geo/ResourceManager.java`, `api/items/ItemSetting.java`, `api/player/PlayerBackpack.java`, `api/player/PlayerProfile.java` (all under `src/main/java/io/github/thebusybiscuit/slimefun4/`), plus `src/main/java/me/mrCookieSlime/Slimefun/api/inventory/BlockMenu.java`, `src/main/java/me/mrCookieSlime/Slimefun/api/inventory/UniversalBlockMenu.java`
- Modify (fully-qualified rewrite): `src/main/java/me/mrCookieSlime/Slimefun/api/BlockStorage.java` (3 sites)
- Delete: `src/main/java/io/github/thebusybiscuit/slimefun4/libraries/dough/config/Config.java`, `.../libraries/dough/skins/PlayerHead.java`, `.../libraries/dough/skins/CustomGameProfile.java`

**Interfaces:**
- Consumes: `dev.yanianz.sudo.config.Config` (SuDo 26.2.0 — five constructors: `Config(Plugin)`, `Config(Plugin,String)`, `Config(File,FileConfiguration)`, `Config(File)`, `Config(String)`).
- Produces: no new symbols; the dough `Config`/`PlayerHead`/`CustomGameProfile` types no longer exist.

- [ ] **Step 1: Swap the import in the 19 files**

In each of the 19 files listed above, replace the line
`import io.github.thebusybiscuit.slimefun4.libraries.dough.config.Config;`
with
`import dev.yanianz.sudo.config.Config;`.

Mechanical command (macOS/BSD `sed`; run from repo root):
```bash
grep -rl "import io.github.thebusybiscuit.slimefun4.libraries.dough.config.Config;" --include='*.java' src/main \
  | xargs sed -i '' 's#import io\.github\.thebusybiscuit\.slimefun4\.libraries\.dough\.config\.Config;#import dev.yanianz.sudo.config.Config;#'
```
Then confirm: `grep -rn "libraries.dough.config.Config" --include='*.java' src/main | grep "^.*import"` → no output.

- [ ] **Step 2: Rewrite the 3 fully-qualified references in `BlockStorage.java`**

`me/mrCookieSlime/Slimefun/api/BlockStorage.java` uses the dough type fully-qualified (no import) at lines 252, 277, 611. Replace every occurrence of
`io.github.thebusybiscuit.slimefun4.libraries.dough.config.Config`
with
`dev.yanianz.sudo.config.Config`:
```bash
sed -i '' 's#io\.github\.thebusybiscuit\.slimefun4\.libraries\.dough\.config\.Config#dev.yanianz.sudo.config.Config#g' \
  src/main/java/me/mrCookieSlime/Slimefun/api/BlockStorage.java
```
The three sites become e.g. `dev.yanianz.sudo.config.Config cfg = new dev.yanianz.sudo.config.Config(file);` and `new BlockMenu(preset, l, new dev.yanianz.sudo.config.Config(file))`.

- [ ] **Step 3: Verify no dough-Config references remain**

Run: `grep -rn "libraries.dough.config" --include='*.java' src/main`
Expected: no output (all callers now use SuDo; the dough Config file is about to be deleted).

- [ ] **Step 4: Delete the three shim files**

Run:
```bash
git rm src/main/java/io/github/thebusybiscuit/slimefun4/libraries/dough/config/Config.java \
       src/main/java/io/github/thebusybiscuit/slimefun4/libraries/dough/skins/PlayerHead.java \
       src/main/java/io/github/thebusybiscuit/slimefun4/libraries/dough/skins/CustomGameProfile.java
```
(The skins files have zero importers — dead code; the four `PlayerHead.getItemStack`/`setSkin`
call sites already import `dev.yanianz.sudo.skins.PlayerHead`.)

Confirm no lingering references:
`grep -rn "libraries.dough.skins" --include='*.java' src/main` → no output.

- [ ] **Step 5: Build + tests**

Run: `./gradlew build`
Expected: BUILD SUCCESSFUL; all existing tests pass.

If a caller fails to compile because it referenced a member the dough subclass added (it adds
none) or relied on the dough type in an `instanceof`/cast (none found), resolve against the SuDo
`Config` API. If a genuinely unexpected break appears, STOP and report BLOCKED with the compiler
error.

- [ ] **Step 6: Commit**

```bash
git add -u src/main
git commit -m "$(cat <<'EOF'
refactor: repoint dough Config to SuDo; delete dead dough skins shims

dough.config.Config was a pure add-nothing subclass of dev.yanianz.sudo.config.Config;
repoint all 20 call sites (19 imports + 3 fully-qualified in BlockStorage) to SuDo and
delete it. Also delete the dead dough skins shims (PlayerHead, CustomGameProfile) whose
callers already use dev.yanianz.sudo.skins.* directly. libraries/dough now holds only
ChatColors + CustomItemStack (SF2b-d).

Claude-Session: https://claude.ai/code/session_013YFwH85eQ1mJXua7Hqcu1k
EOF
)"
```

Note: `git add -u src/main` stages the modified + deleted tracked files under `src/main` only
(the import swaps, the BlockStorage rewrite, and the three `git rm`s), avoiding `.gradle/` churn.

---

## Self-Review

**Spec coverage:**
- Config repoint (19 imports + BlockStorage FQN) → Steps 1–2. Delete 3 shims → Step 4. Green build → Step 5.
- Success criteria (no dough.config/skins refs, files deleted, build green, dough down to ChatColors+CustomItemStack) → Steps 3, 4, 5.
- Subtype-dependency risk (no instanceof/cast — verified during brainstorming) → Step 5 contingency.

**Placeholder scan:** No TBD/TODO. Exact file list, exact sed commands, exact grep expectations, complete commit message.

**Type consistency:** `dev.yanianz.sudo.config.Config` used identically everywhere; its five constructors match the dough subclass's five `super(...)` constructors, so every `new Config(...)` call resolves unchanged. No signature drift.

**Scope check:** one mechanical task ending in a green build; `CustomItemStack`/`ChatColors` untouched (later slices).
