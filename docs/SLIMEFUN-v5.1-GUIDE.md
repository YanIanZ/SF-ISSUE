# Slimefun v5.1 — Feature Guide & Changelog

This is the **official v5.1** build of this Slimefun fork, targeting **Paper API 26.2** and
**Folia** (tested on SourbyCraft / Luminol 26.2). Everything below is bundled in
`Slimefun-v5.1.jar`.

- **Colors:** 100% hex MiniMessage (`<#RRGGBB>`) — no legacy `§`/`&` codes anywhere.
- **Folia-safe:** every world/entity/inventory operation runs on the region that owns it.
- **Branch:** integration branch `slimefun/experiment` (the stable `main` is kept frozen).

---

## Table of contents

1. [New machines & gadgets](#1-new-machines--gadgets)
   - [Auto-Fisher](#auto-fisher)
   - [Mystery Box](#mystery-box)
   - [Grappling Hook (reworked in v5.1)](#grappling-hook-reworked-in-v51)
   - [Prospector's Ore Scanner](#prospectors-ore-scanner)
   - [Cooking Station & Buff Dishes](#cooking-station--buff-dishes)
2. [Quality-of-life & UX](#2-quality-of-life--ux)
   - [Guide: Favorites, Craftable, Recently-Viewed](#guide-favorites-craftable-recently-viewed)
   - [Guide: Research Progress screen](#guide-research-progress-screen)
   - [Context HUD (action bar + boss bar)](#context-hud-action-bar--boss-bar)
   - [Crafting-time in recipes](#crafting-time-in-recipes)
   - [Per-machine status holograms](#per-machine-status-holograms)
   - [Startup banner (v5.1)](#startup-banner-v51)
3. [Admin Panel (`/sf admin`)](#25-admin-panel-sf-admin)
4. [Resource pack (SlimefunPack-v5.1)](#3-resource-pack-slimefunpack-v51)
5. [Configuration](#4-configuration)
6. [Admin / build notes](#5-admin--build-notes)
7. [Changelog — fixes](#6-changelog--fixes)

---

## 1. New machines & gadgets

### Auto-Fisher

An **END_GAME electric machine** that automatically fishes while it is touching water.

**How to use**
1. Research **"The Art of Automated Angling"** (`/sf research <you>` or in-guide), then craft it.
2. Place the Auto-Fisher so it **touches water** — the block directly **below** it or any of its
   **4 side** neighbours must be water (or a waterlogged block). No water → it idles and uses no
   energy.
3. Connect it to an **energy network** (it consumes **30 J per catch**, buffer 512 J).
4. It fishes on its own: every **6 Slimefun ticks** it rolls a catch into its output slots.
5. **Optional bait:** put **raw Cod** or **raw Salmon** in an input slot. Bait **halves** the cast
   interval (to 3 ticks) and shifts the odds toward **treasure**, consuming 1 bait per catch.

**Loot** (weighted): mostly common fish (cod / salmon / tropical / pufferfish), occasional
**treasure** (enchanted book, nautilus shell, name tag, saddle, bow) and some junk. Bait raises the
treasure chance.

**Recipe** (Enhanced Crafting Table): fishing rod · prismarine shard · fishing rod / electric motor
· water bucket · electric motor / reinforced alloy ingot · big capacitor · reinforced alloy ingot.

---

### Mystery Box

A **placed, non-electric magical block** — a slot-machine you feed keys.

**How to use**
1. Research **"A Box of Mysteries"**, then craft a **Mystery Box** and some **Mystery Keys**.
2. Place the box and open it. You'll see: an **input slot** (left), a **✦ Spin** button (center),
   and an **output slot** (right).
3. Put a **Mystery Key** in the input slot and click **✦ Spin**.
4. The box consumes one key, rolls a weighted reward into the output, and plays a firework + sound.
   - **Common (70%)** — vanilla resources (iron, gold, redstone, lapis, coal, xp bottles, a diamond,
     emeralds).
   - **Rare (25%)** — Slimefun items (24-carat gold, reinforced alloy, synthetic diamond/emerald,
     magic lumps, damascus steel, gilded iron).
   - **Jackpot (5%)** — a rare Slimefun item **plus** a bonus firework and a **server-wide broadcast**.
5. If the output slot is full, the spin is refused and your key is **not** consumed.

**Recipes** (Enhanced Crafting Table)
- **Mystery Box:** 24k gold · diamond · 24k gold / diamond · ender chest · diamond / 24k gold ·
  diamond · 24k gold.
- **Mystery Key** (yields **2**): gold ingot ring around a tripwire hook.

---

### Grappling Hook (reworked in v5.1)

The classic Slimefun **Grappling Hook** works correctly on Folia, protects you from fall damage —
and as of **v5.1 it is a reusable fishing-rod tool**.

**How to use**
1. Hold the Grappling Hook and **right-click** to fire an arrow.
2. When the arrow hits a block (or entity), you're **yanked toward it** — short hops pull you in,
   longer shots launch you in an arc.
3. On landing you get **fall-damage immunity** for a few seconds, so grappling up cliffs is safe.

**v5.1 rework**
- The item is now a **fishing rod** (was a lead) and is **reusable**: `consume-on-use` defaults to
  `false`, so the hook is never eaten on use. (Set `consume-on-use: true` in the item's settings to
  restore the old consumable behaviour.)
- **No more lead drops:** the hook's visual leash (a bat leashed to the arrow) dropped a *lead item*
  on the ground when the leash broke — vanilla behaviour, now suppressed for grappling bats only.
- **No more item loss:** grappling into open air (arrow never lands) used to delete a consumed hook
  forever; the despawn path now returns it.
- `despawn-seconds` is honoured as **seconds** (it was scheduled as ticks — hooks despawned 20× early).
- Quitting mid-grapple cleans up the arrow + bat (an invisible, invulnerable bat used to linger).

> Fixed in v5.0: the hook crash-spammed the console on Folia (entity work ran on the wrong region)
> and its fall-damage immunity never actually triggered. Both resolved.

### Prospector's Ore Scanner

A rechargeable exploration tool that finds ores for you — no digging blind.

**How to use**
1. Research **"Prospecting"**, then craft the **Ore Scanner** (a spyglass).
2. Charge it (it's a rechargeable item), then **right-click** to scan.
3. Every ore in a cube around you gets a **colour-coded particle ping** (aqua diamond, maroon
   ancient debris, yellow gold, red redstone, …) so you can see exactly where to dig.
4. An **action-bar tally** lists what was found, **rarest first**:
   `⛏ 2 Diamond · 1 Ancient Debris · 14 Iron`.

Each scan costs a little charge and has a short cooldown. The scan radius is a per-item setting
(default 8, up to 12). Works for all ores — including deepslate and nether variants and ancient
debris (which merge into one tally line each).

---

### Cooking Station & Buff Dishes

An **electric kitchen machine** (buffer 512 J, 10 J/tick) that combines two food ingredients into a
**Dish** that grants potion buffs when eaten. Research **"Cooking"** to unlock the station and all
four dishes.

**How to use**
1. Place the Cooking Station (a smoker) and power it from an energy network.
2. Put the two ingredients of a recipe into the input slots — it cooks for ~10 s per dish.
3. Eat the dish: it restores hunger **and** applies its buffs.

| Dish | Ingredients | Hunger | Buffs |
|------|-------------|--------|-------|
| **Hearty Roast** | Cooked Beef + Carrot | 6 | Regeneration II (5 s) |
| **Spicy Wings** | Cooked Chicken + Nether Wart | 4 | Fire Resistance (15 s) + Speed (10 s) |
| **Golden Feast** | Golden Carrot + Cooked Porkchop | 6 | Absorption II (30 s) + Strength (15 s) |
| **Fisherman's Platter** | Cooked Salmon + Kelp | 5 | Water Breathing + Night Vision (30 s each) |

The guide shows each recipe as ingredient → dish pairs, with the usual crafting-time line.

---

## 2. Quality-of-life & UX

### Guide: Favorites, Craftable, Recently-Viewed

The Slimefun Guide gained a header toolbar and three views:

- **★ Favorites** — **Shift + Right-Click** any item in the guide to favorite/unfavorite it. The
  favorites button (with a live count) opens a paginated list of just your favorites.
- **Craftable now** — a view of items you've unlocked, sorted so the ones you **have the materials
  for** float to the top (with a "have materials" badge).
- **🕓 Recently-Viewed** — the last 18 items you looked at, most-recent first.

Header slots: `menu/back · favorites (+count) · recently-viewed · progress · craftable · search · wiki`.

### Guide: Research Progress screen

A new **🧪 Research Progress** button in the guide header (XP-bottle icon) opens a screen showing:

- **Overall:** `Research: X / Y unlocked (Z%)` with a 10-segment bar (`███░░░░░░░`).
- **Per category:** every item group with its own `unlocked / total (%)` + mini bar — so you can see
  exactly where your progression stands.

### Context HUD (action bar + boss bar)

An **automatic HUD** shows contextual info as you play — no command needed:

- **Look at a Slimefun machine that stores energy** → a **boss bar** appears: it **fills with the
  machine's charge**, and its title shows `⚙ name · ⚡ charge / capacity J · ⚒ crafting %`. The bar
  colour reflects state — **red** (out of power), **blue** (crafting), **green** (charged).
- **Look at any other Slimefun block** → its name in the **action bar**.
- **Hold a Slimefun item** → its name (and charge, if rechargeable) in the **action bar**.
- **Nothing relevant** → the HUD stays empty (no spam).

Toggle it for yourself with **`/sf hud`**; server owners can disable it (or just the boss bar)
globally — see [Configuration](#3-configuration). It's Folia-safe (each player's HUD runs on their
own region).

### Crafting-time in recipes

When you view a machine's recipes in the guide, each output now shows a
**`⏱ Crafting Time: X s`** line, so you know how long that machine takes to produce it.

### Per-machine status holograms

Electric machines now float a **status hologram** above themselves:

- **`⚙ Crafting X%`** while an item is being processed (live progress).
- **`⛔ Not enough energy`** when the machine is starved of power.
- **hidden** when idle.

Updates are Folia-safe and only re-drawn when the text changes, so idle machines cost nothing.
Server owners can turn this off — see [Configuration](#3-configuration).

### Startup banner (v5.1)

The console boot sequence got a facelift:

- A **gradient ASCII "Slimefun" banner** prints at startup with the version, build branch
  (official) and the **detected platform — Paper or Folia — plus the Minecraft version**.
- The post-load summary is a clean, colored block: platform line, **item / research counts**, and
  the project links.
- Platform detection is now accurate (see the v5.1 changelog — `isFolia()` used to report Folia on
  every modern Paper server).

### 2.5 Admin Panel (`/sf admin`)

A management GUI for staff (permission `slimefun.command.admin`, default op).

- Run **`/sf admin`** to open a **paginated, name-sorted** grid of online players shown as their
  **skin heads** (prev/next buttons appear when there are more than 27 players — added in v5.1).
- **Click a player** to open their action menu:
  - **Unlock all research** / **Reset all research**
  - **Give Slimefun Guide**
  - **Teleport to player** (Folia-safe `teleportAsync`)
  - **View stats** (research progress)
  - Back to the list
- v5.1: every action sends the admin a **confirmation message**, and actions on a player who just
  logged out are refused with a clear "no longer online" notice (instead of acting on a ghost).

---

## 3. Resource pack (SlimefunPack-v5.1)

The optional **SlimefunPack-v5.1.zip** gives the nine v5.0/v5.1 items custom textures: Auto-Fisher,
Mystery Box, Mystery Key, Ore Scanner, Cooking Station, and the four dishes.

**How it works** — Slimefun stamps each item with a **CustomModelData** value (from
`plugins/Slimefun/item-models.yml`); the pack uses Minecraft's modern item-model definitions
(`assets/minecraft/items/<base>.json`, `range_dispatch` on `custom_model_data`) so only items
carrying the matching value render the custom model. Vanilla items are untouched — the fallbacks are
copied **verbatim from the 1.21.11 client jar**, so special renderers (ender chest, spyglass-in-hand)
keep working.

**Install (two steps)**
1. Enable the pack on the client (drop the zip into `resourcepacks/` — or host it and set
   `resource-pack:` in `server.properties`).
2. Merge the CustomModelData values (5001–5009, shipped as `item-models.yml.snippet` inside the
   pack) into `plugins/Slimefun/item-models.yml`, then restart.

Targets **pack_format 75** (MC 1.21.11 / SourbyCraft 26.2), with a permissive `supported_formats`
range of 64–99. The textures are generated 16×16 sprites — replace the PNGs under
`assets/slimefun/textures/item/` (same file names) to use your own art.

---

## 4. Configuration

New / notable options in `config.yml` under `options:`:

| Option | Default | Meaning |
|--------|---------|---------|
| `machine-holograms` | `true` | Floating status holograms above electric machines. Set `false` to disable (e.g. for performance on very large machine farms). |
| `hud-enabled` | `true` | The automatic context HUD (looked-at block / held item). Players can also toggle it for themselves with `/sf hud`. |
| `hud-bossbar` | `true` | Show a boss bar (charge fill + crafting %) when looking at a Slimefun machine that stores energy. If `false`, machines fall back to the action-bar HUD. |
| `auto-update` | `false` | Auto-updater. Disabled by default on this fork — it is wired to a custom update server that is not yet enabled, so it never phones home. |

Metrics/analytics are **off** on this build (it is a custom fork, not an upstream Slimefun build), so
there is no bStats phone-home.

---

## 5. Admin / build notes

- **Requires** `folia-supported: true` (already set in `plugin.yml`). Runs on Paper 26.2 and Folia
  forks (Luminol / SourbyCraft 26.2).
- **Version reporting:** a clean release version (e.g. `5.1`) is recognised as an **official custom
  build** — no "unofficially modified build" warning — but it is **not** an upstream build, so it
  never contacts upstream metrics or the upstream auto-updater.
- **Skins:** player-head textures (item and block heads) use the Bukkit `PlayerProfile` /
  `Skull#setOwnerProfile` API — no NMS, mapping-independent, Folia-safe.
- **Build:** `./gradlew clean build` (Java, Gradle Kotlin DSL); the SuDo library is consumed from
  mavenLocal and shaded/relocated into the jar. Deployable artifact: `build/libs/Slimefun-v5.1.jar`.

---

## 6. Changelog — fixes

### v5.1 — full-feature audit (Paper + Folia)

Three parallel deep audits swept every new feature and the Paper/Folia scheduling layer; 20+ bugs
found and fixed:

**Critical**
- **Mystery Box was non-functional:** its input/output slots were "preset" slots — the decorative
  panes were cloned into every menu, so the key could never be inserted, and breaking the box
  dropped UI panes as real items. Fixed; the box now works as designed.
- **Folia thread-safety:** four maps shared across region threads (open-menu registry, per-menu
  click handlers, hologram cache, bow projectiles) were plain HashMaps → now concurrent. These
  could corrupt silently under load on Folia.
- **Folia detection was wrong (SuDo):** `isFolia()` probed a scheduler class that plain Paper has
  shipped since 1.20.1 — every modern Paper server was treated as Folia. Now probes a
  Folia-exclusive class; plain Paper uses the straight Bukkit scheduler again.

**Important**
- Breaking a machine **mid-craft silently deleted the consumed ingredients** — the in-flight
  recipe's results now drop with the block (all AContainer machines, incl. Cooking Station).
- A machine whose block inventory failed to load NPE'd every tick until the error-reporter
  **destroyed the machine and its contents** — now skipped safely; the errored-block terminator
  also ran on the wrong thread on Folia (block never cleared).
- The **shutdown block-data flush** could silently no-op if a ticker cycle was mid-flight →
  bounded-wait flush; ticker state flags made volatile.
- **Guide "Craftable now"** re-scanned the player's whole inventory O(n log n) times per click
  (inside the sort comparator) → evaluated once per item.
- **Favorites / recently-viewed** did a synchronous full-YAML disk write on every guide click →
  now dirty-flagged and saved with the profile (auto-save/logout).
- **Admin panel** actions held live Player references from when the GUI was built — acting on a
  player who logged out lost items / teleported to ghosts. Handlers now re-resolve by UUID.
- **HUD boss bars** stuck frozen on players' screens through `/reload` (now hidden on disable and
  on runtime toggle-off); identical action-bar resends halved.
- SuDo entity-scheduled tasks were **silently dropped** when the entity was removed first (now run
  their cleanup anyway, matching Paper semantics), and `runAtLocation`/`runAtEntity` returned fake
  task handles (no-op cancel).

**Minor**
- Ore Scanner cooldown map grew forever (now pruned) · Cooking Station guide columns misaligned
  (now strict pairs) · ticker enable/disable race could permanently stop a machine ticking (atomic
  now) · version parser accepted junk like `R0.1-SNAPSHOT` · `InvUtils.hasEmptySlot` was inverted.

### v5.0 line

Everything fixed in the v5.0 line:

**Folia region-safety**
- **Synchronized machine tickers** (Auto-Fisher + every growth accelerator) were dispatched on the
  region-less global thread → they read/wrote the world off the block's region and would thread-crash
  on Folia. Now dispatched on the block's own region — these machines actually work on Folia.
- Research-unlock **fireworks** spawned on the wrong thread → now region-scheduled.
- **Grappling Hook** scheduled entity work on the region-less global thread → crash spam; now runs
  on the arrow's region, teleports use `teleportAsync`, and its shared maps are concurrent.
- **Machine status holograms** / `HologramsService` touched ArmorStands off-region (it keyed off
  `Bukkit.isPrimaryThread()`, which is false on Folia region threads) → now checks region ownership
  and schedules onto the hologram's region.
- **Admin panel** touched a selected player's inventory/location/health from the admin's region → now
  hops to that player's region for the guide-give + teleport (skin heads show name only).
- **Shutdown ticker** rescheduled a task while the plugin was disabling → `IllegalPluginAccessException`
  on stop; now skips rescheduling once disabled.

**Gameplay / exploits**
- **Mystery Box dupe:** spinning against a near-full output slot deposited a partial reward but kept
  the key, letting players farm free items. Now the spin only proceeds when the reward fully fits.

**Rendering / colors**
- `/sf help` printed raw `TextComponentImpl{...}` → now rendered.
- Recipe-type lore (e.g. Multiblock), rechargeable **charge** lines and the **Multi Tool** "Mode:"
  line showed literal `<gray>` / `<#hex>` tags → now rendered.
- Contributor roles (Artist, Maintainer, …) showed `Missing string guide.credits.roles.<role>`
  after the hex migration → fixed.
- **Skull NMS** startup crash on Mojang-mapped 26.2 (`TileEntitySkull` not found) → replaced with
  the Bukkit `Skull` API.

**Other**
- Marked the build as an **official v5.0** release; disabled upstream auto-update/metrics.
- Added **iYanZ** as a Maintainer contributor.
