# Scrap Runner — Claude Design Brief

**Doc version:** 1.0
**Game version:** v0.9.0
**Audience:** Claude Design
**Output target:** SVG assets + HTML/CSS HUD, exportable to Claude Code for integration.

---

## 0. What I need from you

Scrap Runner currently draws all its sprites *procedurally in canvas code* — clean but flat. This brief asks for two things:

1. **A HUD/UI visual pass** — design the in-play HUD and screen chrome as polished SVG/HTML/CSS.
2. **Improved SVG assets** for every ship, enemy, boss, weapon glyph, pickup, status, and relic — distinct, on-brand silhouettes that can replace (or be rasterised from) the procedural sprites.

**Important capability note:** you output **vector (SVG) and HTML/React**, not raster images. That is ideal for HUD and for crisp scalable ship/icon art. (Painted/pixel raster sprite sheets, if we later want them, come from a separate image-gen tool — not this brief.) Please deliver clean, layered, **named** SVGs with sensible `viewBox`es and transparent backgrounds, plus HTML/CSS for HUD panels. Where useful, group by layer (silhouette / detail / glow) so Claude Code can tint or animate parts.

---

## 1. Art direction

**Mood:** melancholic salvager, neon-cyberpunk, lonely-but-beautiful void. Think weathered, scavenged hardware lit by neon — not glossy military sci-fi. Restraint over spectacle.

**Grammar (please honour):**
- **Palette discipline.** Teal/cyan = structure. Magenta = accent. **Amber = currency only** (don't use it decoratively). Status colours are reserved (below). Most assets read in teal + magenta.
- **Rationed glow.** Selective neon glow on edges/cores, not a uniform bloom. Dark interiors, bright rims.
- **Angular, notched, "HUD-kit" framing** — cut corners, corner brackets, thin scanlines, chevron accents. The existing frame uses notched/cut-corner rectangles and corner brackets; match that language.
- **Silhouette-first.** Each ship/enemy/boss must be identifiable by silhouette alone at small size (entities render ~24–48px tall in play).

### Exact palette (hex)
| Role | Hex |
|------|-----|
| Structure (teal/cyan) | `#38e8ff` |
| Accent (magenta) | `#ff2da0` |
| Currency (amber) | `#ffd166` / `#ffae3b` |
| Health / owned (green) | `#5ce28a` |
| Charge (purple) | `#c77dff` |
| Heat (orange) | `#ff7849` |
| Gunner / danger (red) | `#ff5470` |
| Player | `#38e8ff` (cyan) |
| Dim / muted | `#5a6a8a` |
| Panel ground | `rgba(8,12,26,0.85)` |

### Status colours (reserved — use only for the matching status/asset)
Burning `#ff8c42` · Shocked `#7aa2ff` · Marked `#ffd166` · Fractured `#ff5470` · Chilled `#aef0ff`.

---

## 2. SVG delivery spec

- **Backgrounds transparent.** No baked-in frames around individual sprites.
- **Player/enemy ships:** `viewBox="0 0 64 64"`, ship **facing up** (nose at top), centred, occupying ~80% of the box. Provide a version with glow and one without (Claude Code may add glow at runtime).
- **Bosses:** `viewBox="0 0 160 160"`, facing down toward the player (nose/menace pointing down), centred.
- **Icons/glyphs** (weapons, statuses, relics, upgrades): `viewBox="0 0 32 32"`, mono-line where possible so they can be tinted by CSS `currentColor`.
- **Layering:** group as `<g id="silhouette">`, `<g id="detail">`, `<g id="glow">` so parts can be recoloured/animated. Use `stroke` for outlines (1.5–2 units), darker fills for interiors.
- **Naming:** `ship_<id>.svg`, `enemy_<id>.svg`, `boss_<id>.svg`, `weapon_<id>.svg`, `status_<id>.svg`, `relic_<id>.svg`, `pickup_<id>.svg`. IDs listed below.
- Keep node counts modest (these animate at 60fps once integrated).

---

## 3. Ships — all 8 hulls (`ship_<id>`)

Each is a *scavenged* salvager hull with a distinct role and silhouette. Cyan structure, magenta accent, dark interior, glowing engine.

| id | name | identity / silhouette direction |
|----|------|--------------------------------|
| `standard` | Standard | balanced arrowhead; the baseline scavenger. |
| `wake` | Wake | speed/dash hull — sleek, swept-back, twin trailing fins (leaves an engine trail). |
| `reliquary` | Reliquary | ornate, almost sacred — carries relics; add a small shrine/core motif. |
| `choirmaster` | Choirmaster | drone-carrier — bulkier body with side bays; reads as a small mothership. |
| `scrapwright` | Scrapwright | the junker — asymmetric, bolted-together plates, exposed salvage. |
| `reaver` | Reaver | aggressive, bladed, forward-leaning — predatory. |
| `aegis` | Aegis | defensive — broad, armoured prow, shield emitters; reads "tank". |
| `vanguard` | Vanguard | military-surplus, clean and angular — the most "finished" hull. |

For each, the existing code draws a unique procedural shape via a `drawHullSprite` branch; the SVG should *improve* on that identity, not contradict it. Engine glow at the rear is shared. Aegis should clearly suggest shielding; Wake clearly suggests speed; Choirmaster clearly suggests bays.

---

## 4. Enemies (13) & bosses (6)

### Enemies (`enemy_<id>`) — small, hostile, mostly facing down
Distinct silhouettes, danger-tinted where relevant:
`drifter` (basic), `weaver` (sine-mover, sleek), `gunner` (has a visible barrel — it telegraphs shots), `tank` (slow armoured block), `swarmling` (tiny, swarms), `brute` (heavy, spread-shooter), `carrier` (spawns/launches — bays), `shielddrone` (emits shield links — orb/emitter look), `repair` (medical/repair motif), `leech` (organic, grabby — drains charge), `assassin` (sharp, teleporting — glitchy/phase look), `artillery` (long-range, sits high — cannon/mortar), `mimic` (deceptive — should look *almost* like a pickup or ally).

### Bosses (`boss_<id>`) — large, menacing, themed colour
| id | name | shape direction | colour |
|----|------|-----------------|--------|
| `warden` | The Warden | hexagonal fortress | `#ff5470` |
| `choir` | Carrion Choir | organic lobed cluster | `#c77dff` |
| `dread` | The Last Dreadnought | heavy angular wedge/arrowhead | `#ff4d6d` |
| `sun` | The Hollow Sun | spiked sunburst/star | `#ff8c42` |
| `node` | Tempest Node | angular crystal with orbiting nodes | `#7aa2ff` |
| `hive` | Hive Sovereign | wide hive-carrier with hex cells | `#5ce28a` |

All bosses share a glowing central **eye/core** motif. Each must read distinctly from its silhouette even before the colour registers.

---

## 5. Weapon glyphs (12, `weapon_<id>`) + projectile treatments

Mono-line 32×32 icons, tintable, used in the armoury/codex/HUD. Group by element so they read as families:
- **Kinetic:** `repeater`, `scatter` (Scattergun), `rail`, `splitshot` (Split Shot).
- **Thermal/burn:** `torch` (beam), `missilepod` (Missile Pod), `mortar` (Mortar Lobber), `seekerpod` (Hunter Seekers). Incendiary — flame/ember accents.
- **Shock:** `witchlight` (chain), `arclance` (Arc Lance, beam), `tempest` (Tempest Coil, chain). Lightning accents.
- **Cryo:** `rimelance` (Rime Lance), `frostfan` (Hailburst shotgun). Frost/crystal accents.

Also, if you're up for it, **projectile/beam/chain visual treatments** as small reusable SVGs: a missile/rocket with flame trail, a beam segment, a chain-arc segment, a cryo slug, a pellet. These map to in-game bullet rendering.

---

## 6. Pickups, statuses, relics, orbitals (icon sets)

- **Pickups (`pickup_<id>`):** `scrap` (amber shard — currency), `health`/repair (green cross), `pu_charge` (purple bolt/cell), `pu_frenzy` (orange), `pu_orbital` (orbit ring), `wreckage` (overcharge salvage), `cores` (magenta gem — premium currency).
- **Statuses (`status_<id>`):** burning (flame), shocked (arc), marked (target reticle), fractured (cracks), chilled (frost) — in their reserved colours.
- **Relics (`relic_<id>`):** 9 — `ashenheart`, `conduction`, `brittlecold`, `quarrybrand`, `faultline`, `embertally`, `glasscannon`, `vampcore`, `siege`. Small, talisman-like, tinted to their status/theme. These are "rare drops" — they should feel precious.
- **Orbitals (`orbital_<id>`):** `guard` (shield orb, `#7fd9ff`), `gun` (`#ffd166`), `side` (`#ff5fa0`).
- **Crew portraits (5):** Navigator, Weapons Officer, Bosun, Shield Tech, Engineer — abstract role-insignia portraits (no faces needed; iconographic is fine), in panel style.

---

## 7. HUD & screen chrome (HTML/CSS + SVG)

The HUD reads as a **targeting computer**. Please design these, keeping the cut-corner/bracket/scanline grammar. Note the current pain points so the redesign solves them.

### In-play HUD (overlaid on the 640×720 play-field)
- **Top-left telemetry:** score, salvage (with currency icon), mod pips. *Pain point:* a long score collided with the mod pips — give the cluster a clear two-row layout that tolerates 6-digit scores.
- **Top-centre:** a title tab + a **threat readout** (target count / boss-contact warning) below it. *Pain point:* these were overlapping — keep them clearly stacked.
- **Top-right:** LEVEL + two paired **dials** (charge gauge, ability cooldown). *Pain point:* felt cramped against the edge — give breathing room.
- **Right rail:** transient state pips (overcharge, frenzy, drones, phoenix).
- **Bottom:** hull-integrity strip (with shield pips + optional armour/heat sub-bars), and an XP scan-rail with RANK.
- **Edge scrims:** soft top/bottom gradients so game objects don't bleed over the readouts (already added in code — your design should assume a darkened band top & bottom).
- **Action buttons:** DASH (bottom-left), SPECIAL (bottom-right), central PAUSE — chunky, glowing when ready.

### Side legend panels (HTML/CSS, ~300px wide, shown ≥1280px)
Two flanking panels (left = controls/loadout/ability/mods; right = telemetry/status-effects/reactions). Cut-corner clip-path, scanline overlay, neon header rules, animated equaliser/telemetry flourishes. Keep reaction rows to **one line each**.

### Screens (full-canvas)
Design cohesive treatments for: **menu**, **draft** (mod-choice cards), **pause = Loadout Dossier** (two-column build summary), **codex**, **career telemetry** (bar charts), **relic vault**, **crew quarters** (crew cards with rank/XP/trait), **game-over** (run analytics). Reusable components welcome: panel, bar, gauge, bracket, card, header rule, button.

---

## 8. Priorities

If delivering in stages:
1. **HUD in-play band** (top telemetry/threat/dials) + the integrity/XP strip + action buttons — highest visible impact.
2. **The 8 ship SVGs** — the player stares at theirs constantly.
3. **Enemy + boss silhouettes.**
4. **Icon sets** (weapons, statuses, relics, pickups).
5. **Screen chrome** (draft cards, dossier, vault, crew quarters).

Establish a small **design system** first (colours above, a couple of panel/bar/button components, the bracket/scanline motifs) so everything inherits it. Then iterate per asset. Export the system + assets to Claude Code so they can be wired into the canvas/HTML.
