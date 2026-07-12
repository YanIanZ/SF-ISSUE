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
| 📖 Guide | **Favorites** (Shift+Right-Click), **Craftable-now**, and **Recently-Viewed** views. |
| 🎨 Colors | 100% hex MiniMessage — no legacy `§`/`&` codes anywhere. |
| 🧵 Platform | Fully **Folia-safe** (region-scheduled world/entity/inventory access) on Paper 26.2. |

See the [feature guide](docs/SLIMEFUN-v5.0-GUIDE.md) for full usage + recipes.

---

## 🗂️ Design docs

Each update has a **spec** (design) and a **plan** (implementation steps).

### Features & gameplay
- Auto-Fisher — [spec](docs/specs/2026-07-11-auto-fisher-design.md) · [plan](docs/plans/2026-07-11-auto-fisher.md)
- Mystery Box — [spec](docs/specs/2026-07-12-mystery-box-design.md) · [plan](docs/plans/2026-07-12-mystery-box.md)
- Guide Favorites & Craftable — [spec](docs/specs/2026-07-11-sf4-guide-favorites-craftable-design.md) · [plan](docs/plans/2026-07-11-sf4-guide-favorites-craftable.md)
- Guide Recently-Viewed & polish — [plan](docs/plans/2026-07-11-sf4b-recent-and-polish.md)

### Platform & migration
- SuDo 26.2 + Paper 26.2 bump (SF1) — [spec](docs/specs/2026-07-10-sf1-sudo-26.2-paper-migration-design.md) · [plan](docs/plans/2026-07-10-sf1-sudo-26.2-paper-migration.md)
- Paper API 100% (SF3) — [spec](docs/specs/2026-07-11-sf3-paper-api-design.md)
- Slimefun ↔ SuDo migration — [spec](docs/specs/2026-06-30-slimefun4-sudo-migration-design.md) · [plan](docs/plans/2026-06-30-slimefun4-sudo-migration-plan.md)
- Maven → Gradle migration — [spec](docs/specs/2026-05-24-maven-to-gradle-star-migration-design.md) · [plan](docs/plans/2026-05-24-maven-to-gradle-star-migration.md)

### Color / hex migration (dough removal, 100% MiniMessage)
- SF2a — dough repoint — [spec](docs/specs/2026-07-10-sf2a-dough-repoint-design.md) · [plan](docs/plans/2026-07-10-sf2a-dough-repoint.md)
- SF2b — item render + hex — [spec](docs/specs/2026-07-10-sf2b-item-render-hex-migration-design.md) · [plan](docs/plans/2026-07-10-sf2b-item-render-hex-migration.md)
- SF2c — remaining hex sweep — [spec](docs/specs/2026-07-11-sf2c-remaining-hex-sweep-design.md) · [plan](docs/plans/2026-07-11-sf2c-remaining-hex-sweep.md)
- SF2d — remove ChatColor — [spec](docs/specs/2026-07-11-sf2d-remove-chatcolor-design.md) · [plan](docs/plans/2026-07-11-sf2d-remove-chatcolor.md)

---

## 🛠️ Build & install

The deployable artifact is `Slimefun-v5.0.jar` (built from the source repository; **not** stored here).

- Requires `folia-supported: true` (set in `plugin.yml`).
- Runs on Paper 26.2 and Folia forks (Luminol / SourbyCraft 26.2).
- Config toggles: `options.machine-holograms` (default `true`), `options.auto-update` (default `false`).

See [docs/SLIMEFUN-v5.0-GUIDE.md § Configuration](docs/SLIMEFUN-v5.0-GUIDE.md#3-configuration) for details.
