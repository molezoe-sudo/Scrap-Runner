# Scrap Runner — Claude Code Handoff

**Doc version:** 1.0
**Game version at handoff:** v0.9.0 (content-complete vertical slice)
**Source file:** `index.html` (a single self-contained HTML file, deployed as-is to Vercel)
**Audience:** Claude Code (and any human reading along)

---

## 0. How to read this document

This is the master reference for the Scrap Runner codebase. It describes **what exists, where it lives, and why it is built the way it is**, plus the open questions and the road ahead. The game is a single HTML file with no build step yet, so "the codebase" is one `<script>` block. Everything below maps to that file.

**Golden rules for working on this project:**

1. **Verify syntax after every edit.** Extract the script and run Node's checker:
   ```bash
   node -e "const fs=require('fs');let h=fs.readFileSync('index.html','utf8');const m=h.match(/<script>([\s\S]*?)<\/script>/);fs.writeFileSync('/tmp/slice.js',m[1]);" && node --check /tmp/slice.js
   ```
   `node --check` only proves the JS *parses* — it cannot run the canvas or judge visuals. There is no automated way to see or play the game; a running browser (or a headless screenshot) is the only visual verification.
2. **British English everywhere** (colour, armour, maths, flavour). Metric units.
3. **The data-driven philosophy is sacred** (see §3). New content is *data + rules*, never hardcoded into game flow. No piece references another by name.
4. **Bump the version** on meaningful changes in the `VERSION` constant. The filename stays `index.html` always — it's deployed live to Vercel and Nexus links straight to it, so the path must never change. Convention: `MAJOR.MINOR.PATCH`; **1.0.0 = first packaged (Tauri/Electron) release**.
5. **Balance values are first-draft placeholders.** Almost every number (damage, drop rates, costs, durations, trait magnitudes) is untuned. See §13.

---

## 1. Vision & aesthetic

A **melancholic-salvager, neon-cyberpunk vertical-scrolling roguelite shmup**. You pilot a scavenger ship "harvesting the void" — endless descent, scaling difficulty, run-based meta-progression funded by salvage. The tone is lonely and atmospheric, not bombastic; the fantasy is *building a synergistic machine* out of scavenged parts and watching it come together.

**Visual grammar:** restraint and palette discipline. Teal/cyan is structure, magenta is accent, amber is *strictly* currency, status colours are reserved. Glow is rationed, not sprayed. The HUD reads like a "targeting computer."

**Long-term arc:** content-complete now → **balance pass** → **art pass** (replace procedural vector sprites with authored raster assets) → **package as desktop app** (Tauri preferred over Electron for size).

---

## 2. Tech, constraints & file layout

- **Stack:** vanilla JS, HTML5 `<canvas>`, Web Audio. No frameworks, no bundler, no dependencies.
- **Logical play-field:** `const W = 640, H = 720;` (portrait). All gameplay coordinates are in this space.
- **Canvas fit:** `resize()` scales the 640×720 field to fit the viewport while preserving aspect ratio, reserving `panelW` for the two HTML side panels (which only show at ≥1280px width). Backing store is `W*dpr × H*dpr` with a `setTransform(dpr,…)`; never assume the canvas is exactly 640 wide in device pixels.
- **Persistence:** `window.storage` (key/value). `loadMeta()` / `saveMeta()`. See §11.
- **No server, no network.** Self-contained.

**File organisation (top to bottom of the single file):**

1. `<style>` — page CSS, the two side **legend panels** (`.panel`, `.panel.r`), HUD-kit helper classes.
2. `<body>` — the two `<aside class="panel">` legends + the `<canvas>`.
3. `<script>`:
   - Header comment + `VERSION`, `W`, `H`, `COL` palette.
   - Canvas + `resize()` + input handlers (keyboard/pointer).
   - **DATA tables** (the content): `HULLS`, `PRIMARIES`, `SPECIALS`, `ENEMY_TYPES`, `BOSSES`, `MODS`, `REACTIONS`, `RELICS`, `GEAR`, `CREW_*`, `BG_THEMES`, `ORBITALS`, `UNLOCK_COST`, `META_DEFAULT`.
   - **Build/recompute/stats engine** (`recompute`, `buildRules`, `stats`, `activeRules`).
   - **Effects** (`burst`, `spawnFloat`, `spawnArc`, `spawnDebris`).
   - **Game-flow** (`launch`, `startWave`, `openDraft`, `bankRun`, `abandonRun`, `gameOver`).
   - **Update loop** (`update`, `updateEnemies`, `updateBoss`, `updateBullets`, `updateDrones`, `updateAllies`, `updateOrbitals`, `updateStrikes`, `updateParticles`, `updatePickups`).
   - **Render** (`draw` dispatches by `state`; `drawEntities`, `drawPlayer`, `drawEnemy`, `drawBoss`, `drawHUD`, `drawFrame`, screen-draw functions).
   - **Main RAF loop** (`frame()`).

---

## 3. The synergy engine (the heart of the game)

Everything orbits a **data-driven rule engine**. Pieces (hulls, primaries, specials, mods, relics) are plain data objects. Behaviour is expressed as **rules** of the shape:

```js
{ on: 'onKill', if: c => c.crit, do: c => { /* mutate game state via shared vocabulary */ } }
```

- **Triggers/events:** `onWaveClear`, `onDash`, `onHit`, `onCrit`, `onKill`, `onScrapCollected`, `onSpecialFired`. Fired from the relevant code paths with a context `c` (usually `{ target, type, crit, burning, … }`).
- `buildRules()` rebuilds `activeRules` (a flat array) from the equipped hull + primary + special + every drafted mod. Synergy is **emergent**: a mod that says "on crit, apply Marked" automatically combines with a relic that says "Marked foes take +40%" without either knowing the other exists.
- `recompute()` rebuilds the `stats` object (damage/fire/pierce/crit/chains/maxHp/speed/charge/etc.) from hull + gear + crew + mods + relic + meta-upgrades. **Call `recompute()` whenever the build changes.**

When adding content, prefer expressing it as a rule on the shared vocabulary rather than special-casing it in the loop.

---

## 4. Damage types, statuses & reactions

### Damage types (4 + 1 delivery method)
| Type | Status applied | Notes |
|------|----------------|-------|
| `kinetic` | none | physical; the blank canvas Marked/Fractured mods write onto |
| `thermal` | Burning | |
| `shock` | Shocked | |
| `cryo` | Chilled | |
| `explosive` | Burning | **explosive is a burn subtype** — AoE-delivered fire. Tracked separately in telemetry but mechanically thermal. |

The mapping lives in `damageEnemy` (single choke-point): `if(type==='thermal'||type==='explosive') applyStatus(e,'burning',…)`, etc. **All damage routes through `damageEnemy`**, so telemetry, status application, crit, shields, relic hooks, and on-kill all happen in one place. `dmgSrc` (a module global) tags the *source* (`primary`/`special`/`reaction`/`dot`/`ability`) by save/restore around the relevant blocks.

### Statuses (5)
Burning (DoT, thermal), Shocked (slow + Witchlight chains between shocked foes), Marked (+25% damage taken), Fractured (armour stripped; +damage, stacks to 5), Chilled (slow that stacks toward a hard freeze; `e.frozen` is the frozen-solid state).

### Reactions (the complete 10-pair matrix)
`REACTIONS` is data-driven; consuming both statuses resolves a reaction (guarded by a `reacting` flag). All C(5,2)=10 pairs exist: **Rupture** (mark+frac), **Overload** (burn+shock), **Immolate** (burn+mark), **Meltdown** (burn+frac), **Conduit** (shock+mark), **Shatter** (shock+frac), **Thermal Shock** (chill+burn), **Superconduct** (chill+shock), **Brittle** (chill+frac), **Deep Freeze** (chill+mark → roots via `e.frozen`).

### Recursion guards
`killDepth` and `resolvingSpecial` bound chains so reaction-kills and shatter-explosions can't infinite-loop. Respect them when adding AoE-on-death effects (e.g. the Faultline relic increments/decrements `killDepth` and caps at depth 3).

---

## 5. Weapons (`PRIMARIES`, 12)

`firePrimary(dt)` handles several **kinds**: `beam` (continuous, overheats with a `heatLock` hysteresis), `chain` (lock-on lightning), `projectile` (single or `pellets`+`spread`), and the missile family `missile`/`bomb`/`homing`.

**Elemental spread (after the v0.9.0 consistency pass):**
- **Kinetic (4):** Repeater, Scattergun, Rail, Split Shot — pure guns, statusless.
- **Thermal/Burn (4):** Torch (beam) + the missile family — **Missile Pod, Mortar Lobber, Hunter Seekers are incendiary**: their projectile *and* explosion deal thermal and apply Burning. Pyre special completes the archetype.
- **Shock (3):** Witchlight (chain), Arc Lance (beam), Tempest Coil (chain).
- **Cryo (2):** Rime Lance (projectile), Hailburst (cryo shotgun).

**Missile pipeline:** missile/bomb/homing spawn `pBullets` with `missile:true`, a `trail`, an optional `home` (homing steer toward nearest live enemy), and an `explode:{r,dmg,type}`. On impact, `explodeBullet(b)` does AoE using `b.explode.type` (defaults thermal). Bombs are slow + huge blast; homing banks toward targets.

**Recent balance fix (v0.9.0):** chain + beam weapons now **only target on-screen enemies** (`e.y>0`) — previously they deleted enemies off-screen before they entered, which made Witchlight a game-breaking auto-clear. Chains also now have **per-hop damage falloff** (`×0.72` per hop): first target full, splash tapers. The fire-rate gating (behind the `player.fire` interval lock) is correct — there is no per-frame multi-hit bug.

---

## 6. Specials (`SPECIALS`, 8)

Fired when Charge is full. `pyre` (burn AoE), `silence` (mark), `nova` (clears enemy bullets + explosive/burn AoE), `purge` (cleanse), `singularity` (gravity pull), `cryonova` (chill pulse), `stasis` (Time-dilation: freezes all enemies ~4s, bosses ~1.2s, via `e.frozen`), `summon` (Wing Support: 4 allied fighters strafe ~5s then leave, via the `allies[]` array + `updateAllies`/`drawAllies`).

Each element now has both weapon **and** special support (burn→Pyre, shock→Nova/arc, cryo→Cryo Nova).

---

## 7. Enemies, elites, champions (`ENEMY_TYPES`, 13)

Drifter, Weaver, Gunner, Tank, Swarmling, Brute, Carrier, **Shield Drone** (shields nearby allies), **Repair** (heals nearby), **Leech** (drains player charge on contact), **Assassin** (teleport-strike via `warpT` telegraph), **Artillery** (off-screen telegraphed AoE strikes via `strikeWarns[]`), **Mimic** (converts statuses you inflict into shielding via `e.copies`).

- Each has a distinct silhouette in `drawEnemy` and an `ai` branch in `updateEnemies`.
- **Edge clamp (v0.9.0):** every non-boss enemy is clamped to `[e.r, W-e.r]` after movement so full sprites stay on-field (sine/repair/shield-drone swing amplitudes were sliding them off the sides).
- **Pre-fire telegraphs (v0.9.0):** gunner/brute/carrier show a charging muzzle (+ dashed aim line for aimed shooters) in the ~0.3s before firing, keyed off `e.fire`.
- **Elites:** gold-ringed variants (wave 4+, chance scales with depth) with type-specific perks (shield bubbles for carrier/tank/brute, faster fire for gunner, +speed otherwise) and 2× rewards.
- **Champions / mini-bosses:** `spawnMiniBoss` at every level ending in 5 (5,15,25…). A giant version of an enemy type using its own AI, `e.champion`+`e.mini` flags, y-capped on-screen, named health bar. *Not* `boss:true` (avoids the `updateBoss` pattern crash).

---

## 8. Bosses (`BOSSES`, 6) & the pattern engine

Big bosses every 10 levels, cycling through the roster (scaled by boss number). **The Warden, Carrion Choir, The Last Dreadnought, The Hollow Sun, Tempest Node, Hive Sovereign.** Each is data: `{name, col, shape, hp, interval, patterns:[...]}`.

- `updateBoss(e,dt)`: intro descent, side-to-side movement, phases by HP %, runs its `patterns` sequence via `bossAttack(e, atk)`.
- `bossAttack` patterns: `radial`, `aimed`, `spiral`, `wall`, `spawn`, plus (v0.9.0) `burst` (aimed cone), `rain` (bullets fall from the top across the field), `seekers` (homing), `cross` (4 rotating arms). All scale with `e.phase`.
- `bossBullet` tints projectiles with `e.col` (themed boss bullets; `eBullet` render falls back to `COL.enemyBullet`).
- `drawBoss` branches on `e.def.shape` for distinct silhouettes (warden/dread/sun/node/choir/hive), with a `warden`-equivalent default.
- **Juice:** boss WARNING siren + banner (`bossWarn`), slow-mo DEATH SCENE with debris (`bossDeathFx`, `spawnDebris`), low-HP alarm + red vignette.
- **Known gap:** the three newer bosses reuse the generic single-body movement; unique *movement* behaviours would be the next layer of identity.

---

## 9. Meta-progression

Two currencies: **salvage** (common, amber, from kills/runs) and **cores** (premium, magenta, drop only from big bosses at level 50+). Cores fund crew exclusively.

- **Hangar** (tabbed: FLEET/ARMING/WORKSHOP) — `UNLOCK_COST`-gated unlocks. Auto-iterates the data tables, so new content appears automatically. `buyUnlock`/`buyMetaUpg`.
- **Dry Dock** — visual ship schematic with 5 system **nodes** (engine/weapon/hull/shield/reactor); `drawDockBay` per slot for SWAP/UPGRADE gear + the Crew Quarters entry.
- **Gear** — 5 slots (`GEAR_SLOTS`), 2 trade-off pieces each, levelled 0–10 with salvage, applied in `recompute`.
- **Crew** — one per maxed (level-10) gear slot. **Two progression layers:** (1) *trained level* paid in Cores (+10%/slot/level), and (2) *earned RANK* from flying. Each crew has `{level, xp, kills, name}`; helpers `genCrewName`/`crewFix`/`crewLevel`/`crewTitle`/`crewTier`/`crewAboard`, role map `CREW_ROLE`, trait map `CREW_TRAIT`. Earned-rank traits (tier = floor(rank/5)) apply in `recompute`: Navigator *Slipstream* (+spd), Weapons Officer *Marksman* (+crit), Bosun *Bulwark* (+hp), Engineer *Overclock* (+charge); Shield Tech *Failsafe* = emergency shield when integrity <12% (in `damagePlayer`, `player.failCD` cooldown). Cosmetics in `drawPlayer` gated by `crewAboard`: Navigator engine trail, Shield Tech aura, Weapons Officer muzzle glow. XP/kills bank per aboard crew in `bankRun`.
- **Relics (9)** — one equipped per run (`META.equippedRelic`, resolved to `relic` at launch). Status set complete: Ashen Heart (burn), Conduction Coil (shock), Brittle Cold (chill), **Quarry Brand** (mark), **Faultline** (fractured); plus Ember Tally (scaling), Glass Reactor (pact), Carrion Core (sustain), **Siege Protocol** (+25% dmg stationary / −10% moving). Hooks live in `damageEnemy`/`killEnemy`. Drop from boss/champion/elite via `grantRelic` (un-owned only). Relic Vault screen + codex section.
- **Orbitals (3)** — `pu_orbital` rare pickup grants a 60s orbiting companion: `guard` (intercepts bullets, ~5 hp), `gun` (fires forward), `side` (fires L+R). `addOrbital`/`updateOrbitals`. Distinct from Carrier drones.
- **Upgrades** — meta `UPGRADES_META` (frame/magnet/charge/coremag…).

---

## 10. HUD, UI & screens

- **In-play HUD** (`drawHUD`): top-left telemetry (score/salvage/mod pips), top-centre threat readout (below the title tab), top-right LEVEL + paired charge/ability dials, right-rail temp states, bottom integrity strip, bottom XP rail. **Recent fix (v0.9.0):** the top band was reflowed to stop overlaps (title tab vs threat readout, score vs mod pips, dials vs LEVEL).
- **Edge scrims (v0.9.0):** `drawFrame` paints soft top/bottom gradient scrims so bullets/enemies don't bleed over the HUD bands.
- **Action buttons** (`drawActionButtons`): DASH/SPECIAL + central PAUSE, wired to `tryAbility`/`tryFireSpecial`. Pointer routing checks `uiButtons` before ship movement.
- **Pause = Loadout Dossier** (`drawPause`): two-column build summary (ship/relic/gear/crew | combat stats/mods/synergy-rule count).
- **Other screens** (dispatched in `draw` by `state`): menu, draft (`drawDraft`), codex (`drawCodex` — auto-iterates all tables + Status Effects + Relics sections), career telemetry (`drawStats`), relic vault (`drawRelics`), crew quarters (`drawCrewQuarters`), game over with run analytics.
- **HUD toolkit:** `cutRect`, `panel2`, `bracket`, `bracketSoft`, `chevrons`, `hudBar`, `hudHeader`, `reticle`, `gauge`, `btn`, plus glyph drawers. Reuse these for consistency.
- **Side legend panels** are HTML/CSS (not canvas), widened to 300px in v0.9.0; reaction rows pinned to one line via `.blk.rx .li{white-space:nowrap}`.

---

## 11. Persistence, telemetry, audio

- **META schema:** `{salvage, cores, maxLevel, unlocks:{}, seen:{}, relics:{}, equippedRelic, gear:{slot:{piece,level,crew}}, upg:{}, life:{}}`. `META_DEFAULT` + migration guards (`||` fallbacks, `crewFix`/`lifeDefault`/`gearIntegrity`). **Always migrate defensively** — old saves must keep working.
- **Telemetry:** per-run `RUN` (`freshRun()`) tracks dmg by type/source, kills, reactions, accuracy. Lifetime `META.life` (`lifeDefault()`) merged in `bankRun`. Surfaced in the game-over analytics panel and the Career screen. Known limits: drone fire counts as 'primary'; DoT/reaction kill attribution is last-active; accuracy is approximate.
- **Audio:** a procedural synthwave engine (`Audio2` IIFE) — lookahead scheduler, Mega-Man-style arp + detuned bass + pad + drums, boss-intensity auto-detect, routed through `musicBus` (M mutes). Mix levels are reasoned defaults, never ear-verified.

---

## 12. Conventions & gotchas

- **`str_replace` over a function boundary can swallow the next `function X(){` header** → "Unexpected token". Re-check adjacent function headers after big inserts.
- **`present_files`** the HTML after changes (the human downloads it).
- The COL palette is the law: teal=structure, magenta=accent, amber=currency only, status colours reserved.
- Konami code on the menu (↑↑↓↓←→←→ B A) = dev cache (+salvage, +cores, all relics) for endgame testing.
- Single-pointer limit on mobile: can't move + tap an action button with one finger (known).

---

## 13. Open balance questions (priority for the balancing pass)

Everything is first-draft. Known hot spots:
- **Witchlight/Tempest** — partially addressed (on-screen targeting + falloff); re-validate, watch shock-slow vs slow tanks.
- **Missile-family burn stacking** — now thermal; Mortar's 96px blast seeding Burning across packs may over-perform.
- **Vent + Nova now apply Burning** (explosive=burn) — Vent self-combos Meltdown; check it's not too strong.
- **Crew trait + training stacking** — a maxed, high-rank crew is a big multiplier across all 5 slots.
- **Faultline** death-shatter (recursive, capped at killDepth 3) and **Quarry Brand** mark-leap can cascade.
- **Stasis** 4s freeze tempo swing; **Wing Support** burst DPS; stacked guard **orbitals**; **Glass Reactor** −40% hp vs burst; **Ember Tally** runaway scaling; cryo control-creep / Deep Freeze stun-lock.
- HP/pattern density on the three new bosses; **rain** density.

---

## 14. Roadmap

1. **Balancing pass** (now) — playtest, tune the §13 list. Consider headless logic tests for the pure maths (damage/reaction calcs).
2. **Art pass** — replace procedural vector sprites with authored **raster assets**. See the companion *Claude Design brief* for HUD/SVG direction. Integration plan below.
3. **Packaging** — wrap as a desktop app (**Tauri** preferred: ~3–10MB vs Electron's ~80–150MB). Last step; needs nothing from the game logic except a stable build.

### Asset-integration plan (for the art pass)
The current sprites are *code* (`drawHullSprite`, `drawEnemy`, `drawBoss` draw paths procedurally). Moving to raster:
1. Add an asset loader (preload images / a sprite atlas) before `launch`.
2. Replace the path-drawing in `drawHullSprite`/`drawEnemy`/`drawBoss`/glyph drawers with `ctx.drawImage` from the atlas, keyed by hull/enemy/boss id, sized to the entity's `r` and facing.
3. Keep the procedural versions behind a flag as a fallback until the full set is in.
4. SVG assets (from Claude Design) can be rasterised at load or drawn via `Path2D`; raster sprite sheets (from an image-gen tool) drop straight into `drawImage`.

---

## 15. Versioning

- `VERSION` constant in the header; carry it in the filename (`scrap-runner-vX.Y.Z.html`).
- `MAJOR.MINOR.PATCH`. **1.0.0 = first packaged release.** Until then: MINOR for feature/content batches, PATCH for fixes/balance. Current: **0.9.0**.
