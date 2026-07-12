# SF-ISSUE — Slimefun (Paper 26.2 + Folia) fork · Docs & Design Hub

Documentation, feature guides, and design docs for the **official v5.0** build of this Slimefun
fork — targeting **Paper API 26.2** and **Folia** (tested on SourbyCraft / Luminol 26.2),
maintained by **iYanZ**.

> This repository holds **documentation only** — no source code. It tracks every update to the fork:
> what shipped, how to use it, and the spec/plan behind each change.

---

## 📘 Start here

- **[Slimefun v5.0 — Feature Guide & Usage](docs/SLIMEFUN-v5.0-GUIDE.md)** — every new feature, how to
  use it, config options, and the full fix changelog.
- **[Folia notes](docs/FOLIA.md)** — Folia region-safety rules this fork follows.

---

## ✨ What's new in v5.0 (highlights)

| Area | Update |
|------|--------|
| ⚙️ New machine | **Auto-Fisher** — auto-fishes while touching water; optional cod/salmon bait boosts speed & treasure. |
| 🎁 New block | **Mystery Box** — insert a Mystery Key, spin for a weighted reward (vanilla / Slimefun / jackpot). |
| 🪝 Gadget | **Grappling Hook** — fixed on Folia + fall-damage immunity now actually works. |
| 🖥️ UX | **Per-machine status holograms** (`⚙ Crafting X%` / `⛔ Not enough energy`), toggleable. |
| ⏱️ UX | **Crafting-time** shown on every machine recipe in the guide. |
| 📖 Guide | **Favorites** (Shift+Right-Click), **Craftable-now**, **Recently-Viewed**, and **Research-Progress** views. |
| 🧪 Guide | **Research Progress** screen — overall completion + per-category bars. |
| 🔎 HUD | **Automatic context HUD** — a **boss bar** (charge + crafting %) for the machine you look at, action-bar info for other blocks / held items (`/sf hud`). |
| 🛠️ Admin | **`/sf admin` panel** — online-player skin heads → unlock/reset research, give guide, teleport, stats. |
| 🎨 Colors | 100% hex MiniMessage — no legacy `§`/`&` codes anywhere. |
| 🧵 Platform | Fully **Folia-safe** (region-scheduled world/entity/inventory access) on Paper 26.2. |

See the [feature guide](docs/SLIMEFUN-v5.0-GUIDE.md) for full usage + recipes.

---

## 🛠️ Build & install

The deployable artifact is `Slimefun-v5.0.jar` (built from the source repository; **not** stored here).

- Requires `folia-supported: true` (set in `plugin.yml`).
- Runs on Paper 26.2 and Folia forks (Luminol / SourbyCraft 26.2).
- Config toggles: `options.machine-holograms` (default `true`), `options.auto-update` (default `false`).

See [docs/SLIMEFUN-v5.0-GUIDE.md § Configuration](docs/SLIMEFUN-v5.0-GUIDE.md#3-configuration) for details.
