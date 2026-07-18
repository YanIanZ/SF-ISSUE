# MiningExpansion — Deep Earth Mining for Slimefun (Paper 26.2 + Folia)

A Slimefun addon built around a **deep-earth mining progression**: dig tiered ores far underground,
crack geodes, cut gems with timed skill, socket them into gear, and grow living crystal. Several
machines use **real-time skill minigames** where timing changes the outcome.

- **Target:** Paper API `26.2` and **Folia** (verified on SourbyCraft / Luminol 26.2)
- **Depends on:** Slimefun v5.1 fork (this hub)
- **Item text:** MiniMessage · **Folia-safe** throughout

Guide categories in the Slimefun guide: **Deep Earth — Ores & Blocks**, **Gems & Materials**,
**Crystal Gear**, and **Machines**.

---

## Progression at a glance

```
Mine tiered ore (deep) ──▶ Raw Crusher ──▶ Slimefun dust
Find geodes ──▶ Geode Cracker (timed) ──▶ tiered loot / rough gems
Rough gems ──▶ Cutting Table (timed strike) ──▶ cut gems (Rough / Fine / Flawless)
Cut gems ──▶ Socket Press ──▶ socketed tools & armor
Crystal Bud (grows over time) ──▶ harvest crystal
```

## Ores & pickaxe tiers

Ores generate deep underground (each has a minimum Y) and are **tier-gated by pickaxe**:

- **Minable with any Pickaxe** — the entry tier
- **Requires a Crystal Pickaxe**
- **Requires a Mithril Pickaxe**
- **Requires an Adamantite Pickaxe** — the deepest / Mythic tier

Mining the wrong tier tells you which pickaxe you need. Raw ore feeds the **Raw Crusher**, and the
Slimefun **Ore Crusher doubling** applies to its output.

## Machines

| Machine | What it does | Skill element |
|---|---|---|
| **Raw Crusher** | Crushes raw metals into Slimefun dust — 1 raw every 2 seconds | passive machine |
| **Geode Cracker** | Cracks geodes for tiered loot | timed crack — **"PERFECT crack!"** lands the best odds |
| **Cutting Table** | Cuts rough gems into finished gems | **one timed strike** — a miss *shatters* the gem; perfect release = jackpot odds; quality **Rough / Fine / Flawless** |
| **Socket Press** | Sockets cut gems into tools & armor | splits/sets whatever is placed before it |

The **Cutting Table** and **Geode Cracker** are the skill machines — mistime and you lose the
material, nail it and you get the top quality / loot tier.

## Gems & quality

Rough gems are cut at the **Cutting Table** into **Rough / Fine / Flawless** quality. Higher quality
= stronger effect when socketed. Missing the timed strike shatters the gem into a shard ("Remains of
a shattered gem").

## Crystal Gear

A crystal-themed tool/armor set with location-aware effects, e.g.:

- *"Sure-footed below Y-56"* — bonus deep underground
- *"Mines the Mythic depths"* — high-tier mining
- *"A lattice no blade can follow"* / *"Facets that turn every strike"* — defensive facets
- *"Tills with crystalline patience"* — a crystal hoe
- *"Crowned in living crystal"* — a crystal helmet

Socketed **cut gems** modify gear behaviour via the Socket Press.

## Crystal Bud (growth)

A rare crystal formation that **grows over time — harvest when mature**. A renewable source of
crystal materials for the gear/gem line.

## Folia safety

Machines and the timed minigames tick on the correct region via the SuDo scheduler; world-gen ore
population, geode nodes, and growth run Folia-safely. As with any addon here, its plugin.yml
declares `folia-supported: true`.

---

*Part of the [SF-ISSUE](https://github.com/YanIanZ/SF-ISSUE) Slimefun v5.1 fork docs hub. Requires
the fork build (Paper 26.2 / Folia).*
