# Slimefun v5.0 — Feature Guide & Changelog

This is the **official v5.0** build of this Slimefun fork, targeting **Paper API 26.2** and
**Folia** (tested on SourbyCraft / Luminol 26.2). Everything below is bundled in
`Slimefun-v5.0.jar`.

- **Colors:** 100% hex MiniMessage (`<#RRGGBB>`) — no legacy `§`/`&` codes anywhere.
- **Folia-safe:** every world/entity/inventory operation runs on the region that owns it.
- **Branch:** integration branch `slimefun/experiment` (the stable `main` is kept frozen).

---

## Table of contents

1. [New machines & gadgets](#1-new-machines--gadgets)
   - [Auto-Fisher](#auto-fisher)
   - [Mystery Box](#mystery-box)
   - [Grappling Hook (fixed)](#grappling-hook-fixed)
   - [Prospector's Ore Scanner](#prospectors-ore-scanner)
2. [Quality-of-life & UX](#2-quality-of-life--ux)
   - [Guide: Favorites, Craftable, Recently-Viewed](#guide-favorites-craftable-recently-viewed)
   - [Guide: Research Progress screen](#guide-research-progress-screen)
   - [Context HUD (action bar + boss bar)](#context-hud-action-bar--boss-bar)
   - [Crafting-time in recipes](#crafting-time-in-recipes)
   - [Per-machine status holograms](#per-machine-status-holograms)
3. [Admin Panel (`/sf admin`)](#25-admin-panel-sf-admin)
4. [Configuration](#3-configuration)
5. [Admin / build notes](#4-admin--build-notes)
6. [Changelog — fixes](#5-changelog--fixes)

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

### Grappling Hook (fixed)

The classic Slimefun **Grappling Hook** now works correctly on Folia — and actually protects you
from fall damage.

**How to use**
1. Hold the Grappling Hook and **right-click** to fire an arrow.
2. When the arrow hits a block (or entity), you're **yanked toward it** — short hops pull you in,
   longer shots launch you in an arc.
3. On landing you get **fall-damage immunity** for a few seconds, so grappling up cliffs is safe.

> Fixed this release: the hook previously crash-spammed the console on Folia (entity work ran on the
> wrong region) and its fall-damage immunity never actually triggered. Both are resolved.

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

### 2.5 Admin Panel (`/sf admin`)

A management GUI for staff (permission `slimefun.command.admin`, default op).

- Run **`/sf admin`** to open a grid of **online players shown as their skin heads** (each head lists
  gamemode + health).
- **Click a player** to open their action menu:
  - **Unlock all research** / **Reset all research**
  - **Give Slimefun Guide**
  - **Teleport to player** (Folia-safe `teleportAsync`)
  - **View stats** (research progress)
  - Back to the list

---

## 3. Configuration

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

## 4. Admin / build notes

- **Requires** `folia-supported: true` (already set in `plugin.yml`). Runs on Paper 26.2 and Folia
  forks (Luminol / SourbyCraft 26.2).
- **Version reporting:** a clean release version (e.g. `5.0`) is recognised as an **official custom
  build** — no "unofficially modified build" warning — but it is **not** an upstream build, so it
  never contacts upstream metrics or the upstream auto-updater.
- **Skins:** player-head textures (item and block heads) use the Bukkit `PlayerProfile` /
  `Skull#setOwnerProfile` API — no NMS, mapping-independent, Folia-safe.
- **Build:** `./gradlew clean build` (Java, Gradle Kotlin DSL); the SuDo library is consumed from
  mavenLocal and shaded/relocated into the jar. Deployable artifact: `build/libs/Slimefun-v5.0.jar`.

---

## 5. Changelog — fixes

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
