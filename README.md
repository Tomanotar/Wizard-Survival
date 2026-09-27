<div align="center">

# ⚡ WIZARD SURVIVAL

### *Rogue-lite · Action RPG · Multiplayer Co-op*

> 🌐 **Language:** English | [Leer en Español](README.es.md)

<br/>

[![Version](https://img.shields.io/badge/Version-1.0.0--beta-blueviolet?style=for-the-badge&logo=roblox)](https://www.roblox.com)
[![Engine](https://img.shields.io/badge/Engine-Roblox%20Studio-red?style=for-the-badge&logo=roblox)](https://create.roblox.com)
[![Language](https://img.shields.io/badge/Language-Luau-orange?style=for-the-badge)](https://luau-lang.org)
[![Status](https://img.shields.io/badge/Status-Active%20Production-brightgreen?style=for-the-badge)](https://www.roblox.com)
[![Visits](https://img.shields.io/badge/Visits-1.2K%2B-blue?style=for-the-badge)](https://www.roblox.com)
[![Platform](https://img.shields.io/badge/Platform-PC%20%7C%20Mobile-lightgrey?style=for-the-badge)](https://www.roblox.com)

<br/><br/>

### 🎮 **[👉 Click here to play Wizard Survival on Roblox 👈](https://www.roblox.com/es/games/121152223798271/Wizard-Survival)**

<br/>

> **Survive the horde. Master the arcane. Build your legend.**

*Wizard Survival* is a real-time multiplayer cooperative horde-survival RPG set in a dark fantasy world overrun by the undead. Players select, combine, and evolve a deep spell system to fight off endless waves of enemies across three atmospheric maps — each with escalating difficulty, environmental mechanics, and devastating boss encounters.

The game draws from the pick-up-and-play depth of *Vampire Survivors*, the build synergy of *Hades*, and the cooperative tension of classic wave-defense titles, packaged within a hand-crafted client-server architecture that keeps performance tight even with hundreds of entities on screen.

</div>

---

## 📖 Table of Contents

1. [Core Game Loop](#-core-game-loop)
2. [Technical Architecture — Client-Server Split](#-technical-architecture--client-server-split)
3. [Multiplayer Mechanics & Game Design UX](#-multiplayer-mechanics--game-design-ux)
4. [Spell System & Legendary Combinations](#-spell-system--legendary-combinations)
5. [Content Depth — Artifacts & Progression](#-content-depth--artifacts--progression)
6. [Maps, Level Design & World Building](#-maps-level-design--world-building)
7. [Meta-Game, Lobby & Economy](#-meta-game-lobby--economy)
8. [UI/UX & Front-End Polish](#-uiux--front-end-polish)
9. [Development Pipeline & Production Notes](#-development-pipeline--production-notes)
10. [Launch Metrics, Credits & Roadmap](#-launch-metrics-credits--roadmap)

---

## 🎮 Core Game Loop

```
LOBBY  ──►  MAP SELECT  ──►  WAVE SURVIVAL  ──►  BOSS ENCOUNTER
  ▲                                                     │
  │        ┌───────────────────────────────────────┐    │
  └────────┤  Obelisk Chest  ·  Spell Evolution    │◄───┘
           │  Artifact Drops  ·  XP & Level Up     │
           └───────────────────────────────────────┘
```

Each **run** follows this rhythm:

| Phase | Duration | Key Events |
|---|---|---|
| **Early Waves** | 0 – 3 min | Swarm calibration, initial spell selection, mana pool building |
| **Mid-Game Escalation** | 3 – 8 min | Artifact synergies activate, rare spell evolutions unlock, enemy density peaks |
| **Boss Phase** | Triggered at wave thresholds | Ritual Seal arena activates, boss encounters with telegraphed mechanics |
| **Inter-Wave Breather** | ~5s after boss kill | Obelisk Chest descent, spell/artifact choice window, strategic repositioning |

The design goal is a **non-linear power curve**: players feel exponential progression through intelligent build-crafting, not just time investment.

---

## 🔧 Technical Architecture — Client-Server Split

### Spell Lifecycle: Authority & Rendering Separation

The core architectural decision of *Wizard Survival* is a **strict separation of authority**. Every aspect of a spell's execution is divided across two domains to eliminate lag exploitation, prevent desynchronization, and maintain fairness across all connection qualities.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SPELL LIFECYCLE FLOW                                │
├────────────────────────────┬────────────────────────────────────────────────┤
│     CLIENT (Cosmetic)      │          SERVER (Authority)                    │
├────────────────────────────┼────────────────────────────────────────────────┤
│ 1. Player Input detected   │                                                │
│    (mouse click / key)     │                                                │
│         │                  │                                                │
│ 2. Immediate VFX spawn     │ 3. RemoteEvent received                        │
│    Mesh + particle trail   │    Spell type, origin, direction validated     │
│    (predictive, cosmetic)  │         │                                      │
│         │                  │ 4. Hitbox computation                          │
│         │                  │    Spatial algebra + vector math               │
│         │                  │    No visual dependency                        │
│         │                  │         │                                      │
│         │                  │ 5. Damage application                          │
│         │                  │    Humanoid:TakeDamage()                       │
│         │                  │    Cooldown enforcement + anti-exploit         │
│         │                  │         │                                      │
│ 7. Visual impact VFX  ◄────┼─── 6. Hit results dispatched via              │
│    Numbers, screenshake,   │        ReplicatedStorage signal                │
│    audio (all clients)     │        to ALL clients                          │
└────────────────────────────┴────────────────────────────────────────────────┘
```

### Why This Architecture Matters

In a horde-survival game with **80–300 simultaneous enemies** and 4 concurrent players each firing spells at ~2 Hz, the server never processes visual data. Hitbox math runs against pure 3D vectors and spatial queries, making it immune to client frame rate fluctuations or network jitter.

The client's predictive rendering layer means **the player always sees their own spell land instantly**, even on high-latency connections, while the server authoritatively resolves damage without lag exploits.

### Adaptive Rendering & Performance Budget

| Scenario | Optimization Applied |
|---|---|
| Player's own spells | Full VFX at maximum fidelity |
| Allied spells (moderate density) | Simplified particle count, reduced mesh complexity |
| Allied spells (high density: 4 players, mass-fire) | Aggressive culling — particle emitters paused, mesh opacity reduced |
| 200+ enemies on screen | Enemy mesh LOD scaling, staggered AI tick rate on server |

This system ensures the game remains playable on **mobile hardware** without degrading the experience for desktop players.

---

## 👥 Multiplayer Mechanics & Game Design UX

### The Pause Problem — A Design Challenge Unique to Cooperative Survival

In single-player survival games, pausing is trivial. In *Wizard Survival*'s real-time co-op, an individual player's UI interaction (opening a chest, leveling a spell) cannot freeze the game for their teammates. This required designing two distinct interaction modes:

#### Non-Blocking Individual UX

| Interaction | Implementation |
|---|---|
| **Spell Level-Up Selection** | Personal UI overlay with independent countdown timer; enemies dynamically shift aggro to active teammates while the player is in the menu |
| **Individual Chest Opening** | Per-player reward window; other players continue fighting with full awareness |
| **Artifact Selection** | Local menu with a timer; auto-selects a random artifact if the timer expires |

#### Synchronous Consensus Mechanics

| Mechanic | Threshold | Purpose |
|---|---|---|
| **Group Pause** | ≥70% of active players vote "Pause" | Tactical build audit, performance configuration review |
| **Obelisk Chest** | Shared world event | All players see the chest descend simultaneously; rewards are per-player but the opening window is synchronized |
| **Map Vote** | Majority rule on run restart | Ensures group consensus on difficulty and map selection |

### Dynamic Aggro Redistribution

When a player opens any menu, the enemy AI pathfinding system on the server re-evaluates target priorities. Enemies smoothly redirect toward the nearest **active** player, preventing menu abuse as a safe evasion tactic while also protecting players who legitimately need a moment to make a build decision.

---

## 🔥 Spell System & Legendary Combinations

### Base Spell Roster

| Spell | Type | Damage Profile | Key Mechanic |
|---|---|---|---|
| **Disparo Mágico** *(Magic Bolt)* | Projectile | Moderate · Single | Triple-shot spread pattern, fast travel |
| **Bola de Fuego** *(Fireball)* | Area Explosion | High · AoE | Arcing trajectory, ground burst radius scales with level |
| **Rayo Arcano** *(Arcane Beam)* | Piercing Hitscan | Very High · Line | Instant cast, penetrates all enemies in axis |
| **Tormenta de Rayos** *(Thunderstorm)* | Targeted AoE | Extreme · Multi-strike | Chain lightning, stroboscopic impact |
| **Meteoro** *(Meteor)* | Global | Massive · Single Point | Long wind-up, screen-shake, terrain mark |
| **Ventisca** *(Blizzard)* | Zone Control | Low · Sustained | Freeze status, area denial, synergy with water spells |
| **Tsunami** | Displacement | Medium · Push | Enemy routing, knockback chain |
| **Tornado** | Environment | Medium · Sustained | Pulls enemies into vortex, stacking DoT |

### Legendary Fusion System

Spell combinations unlock **Ultimate Fusions** — qualitatively different abilities that cannot be replicated by base spells.

#### ⚡ Láser Longinus *(Singularity Star)*
*Fusion: Meteoro (Asteroid) + Tormenta (Judgment)*

The most powerful ability in the game, executed in three cinematic phases:

```
Phase 1 — Summoning (0.0s – 3.0s)
  · Celestial targeting rings descend from orbit
  · Void disc forms at epicenter, absorbing ambient light
  · Sky globally dims around the impact zone

Phase 2 — Pre-Impact Collapse (3.0s – 4.0s)
  · Rings implode toward singularity at supersonic speed
  · Beam exterior thickens to critical mass diameter

Phase 3 — Singularity Cataclysm (4.0s – 10.5s)
  · 32m-diameter divine beam fires from above
  · Black hole manifests at epicenter
  · Enemies within 60m radius pulled into centripetal vortex and obliterated
  · Gravitational distortion dome expands to 50m
```

#### 🌊 Gran Inundación *(Great Flood Wall)*
*Fusion: Tsunami (Siren's Chant) + Tornado (Extreme Weather)*

A 100m × 20m oceanic wall that emerges from behind the caster and advances linearly at 18 m/s for 7 seconds, physically displacing every enemy in its path with rigid-body knockback simulation.

#### 🔮 Genocidio *(Arcane Genocide)*
*Fusion: Rayo Arcano (Desintegration) + Disparo Mágico (Chain Launch)*

Seven runic seals orbit the caster, firing 32 sequential laser bursts over 4.8 seconds. Each primary impact auto-chains to two additional nearby targets (chain radius: 12m), creating a cascade capable of clearing entire corridors simultaneously.

---

## 💎 Content Depth — Artifacts & Progression

### Artifact System Overview

*Wizard Survival* features **199+ unique functional artifacts** organized into a tiered rarity system. Each artifact modifies one or more of the following dimensions:

- **Base Statistics** — Max HP, movement speed, cooldown reduction, crit chance
- **Spell Behavior** — Projectile count, area radius, damage multipliers, cast frequency
- **Passive Synergies** — Conditional bonuses triggered by spell type, enemy archetype, or player state
- **Build Archetypes** — Enable specific playstyles (Glass Cannon, Barrier Tank, Chain Mage, Time Mage)

### Rarity Tiers

| Tier | Label | Pool Size | Acquisition |
|---|---|---|---|
| ⬜ Common | Standard | ~60 | Wave chests, floor drops |
| 🔵 Rare | Uncommon | ~70 | Obelisk chests, boss rewards |
| 🟣 Epic | Rare | ~40 | Boss kills, Legendary chests |
| 🔴 Legendary | Ultra-rare | ~20 | Late-wave Obelisk, Achievement unlock |
| 🔴 Special Passive | Wild card | ~9 | Low probability at all tiers |

### Notable Artifact Examples

| Artifact | Tier | Effect |
|---|---|---|
| **Lanza de Longinus** | Legendary | Augments Longinus beam duration and collapse radius |
| **Ouroboros** | Legendary | On spell kill: chance to reset the cooldown of a random equipped spell |
| **Excalibur** | Legendary | Melee aura activates when HP < 30%, deals massive area damage |
| **Registros Akáshicos** | Legendary | Passively accumulates a secondary damage stat equal to total lifetime kills |
| **Gaia** | Legendary | HP regeneration scales with number of active players |
| **Necronomicón** | Epic | All spells gain lifesteal proportional to enemies hit per cast |
| **Singularidad** | Epic | Reduces Longinus cooldown, adds gravitational pull to all AoE spells |
| **Puerta Dimensional** | Epic | Teleports projectiles past obstacles |
| **Circuito Espacio-Tiempo** | Special | Slows local time for enemies in a 15-stud radius around the caster |

### Build Crafting Depth

The interaction space between 199 artifacts and 8+ spells (each with multiple evolution paths) creates an enormous emergent buildcrafting system. Intentional design patterns include:

- **Elemental Amplification** — Fire/Lightning artifacts stack multiplicatively with fused elemental spells
- **Chain Reaction Loops** — *Ouroboros* + *Genocidio* can create near-permanent uptime cycles
- **Tank-Mage Hybrids** — *Escudo de Maná* + *Escudo Orgánico* + *Segundo Corazón* makes survivability a damage source

---

## 🗺️ Maps, Level Design & World Building

Three handcrafted maps, each with three distinct difficulty variants that alter lighting, enemy composition, speed, and environmental atmosphere:

### 🌿 El Jardín Ancestral *(Garden of the Ancients)*

Open geometry, readable sightlines — forgiving in layout, relentless in wave volume.

| Difficulty | Atmosphere | Environmental Modifier |
|---|---|---|
| **Fácil** (Easy) | Daytime · clear sky | Standard visibility, baseline enemy parameters |
| **Normal** | Golden hour · 15° sun angle | Long shadows obscure enemy positions; medium speed boost |
| **Difícil** (Hard) | Deep night · `#282C42` | Ground fog + **glowing red eyes as the only enemy visibility cue**; caster's emerald lantern (`#78BE8C`, r=8m) is the sole light source |

### 🏙️ La Ciudad Maldita *(The Cursed City)*

Urban geometry introduces line-of-sight blocking, choke points, and routing decisions absent in open-field combat.

- **Architecture**: Cracked asphalt, ruined skyscrapers fading into upper fog, oxidized vehicles as partial cover
- **Lighting**: Cold desaturated contrast, flickering streetlamps at 2–4 Hz, wet surface reflections
- **Enemy Behavior**: Crawlers navigate low gaps; Brutes smash through obstacles

### 🟡 Las Backrooms *(Liminal Level)*

The most psychologically disorienting map — infinite-corridor repetition creates spatial confusion by design.

- **Architecture**: Beige damp carpet, peeling wallpaper, repeating columns receding to infinity
- **Lighting**: Overhead fluorescents at `#FFF2C6` with 6 Hz sub-flicker; black fog at 45 studs
- **Design Intent**: Claustrophobia is a mechanic — wide AoE spells become survival-critical when flanking is impossible

### Boss Arena — The Ritual Seal

When a boss wave triggers, the map enters **Confinement Mode**:

```
Perimeter Wall (The Cage)
  Dodecagonal energy prism · radius 50m · height 25m
  Crimson translucent energy: #FF0000 · Alpha 0.5 · Emission ×8
  Arcane glyphs scrolling vertically along wall faces

Central Floor Seal
  12-pointed runic star at map center (0, 0, 0)
  Energy lines pulse violet/gold at boss health thresholds

Obelisk Chest (Post-Kill)
  4m obsidian monolith descends in a golden light column
  Settles with physical impact at seal center
```

**Boss Death Sequence:**
1. White screen flash → 1.5s decay
2. All surviving enemies freeze for 2 seconds (server-enforced temporary invulnerability prevents early destruction by residual AoE)
3. At t=2.0s: synchronized mass detonation — ragdoll physics expel limbs and particle debris radially
4. 5-second wave suppression → Obelisk Chest descends

---

## 🏰 Meta-Game, Lobby & Economy

### Dual Economy

| Currency | Source | Use |
|---|---|---|
| 💰 **Monedas** (Coins) | In-run performance, wave completion | NPC shop purchases, temporary run modifiers |
| 💎 **Diamantes** (Diamonds) | End-of-run rating, achievement milestones | Permanent unlocks, lobby upgrades, artifact pool expansion |

### Lobby Structure

```
LOBBY
├── 🧙 Spell Tome NPC     — Unlock base spells, view evolution requirements
├── 🏺 Artifact Vendor     — Browse artifact pool, purchase permanent expansions
├── 📊 Stats Board         — Personal records, leaderboard standings
├── 🗺️ Map Selector        — Choose destination map and difficulty
└── 🎒 Loadout Preview     — Review permanent upgrades and owned unlocks
```

### Persistent Progression

Player advancement persists between sessions via **DataStore**:
- Lifetime currency totals
- Unlocked spell pool
- Artifact collection progress
- Personal performance metrics (max wave, highest damage run, etc.)

---

## 🎨 UI/UX & Front-End Polish

The interface of *Wizard Survival* was built through **manual calibration of every UI property within Roblox Studio** — a deliberate rejection of auto-generated layouts in favor of hand-tuned visual fidelity.

### Design Philosophy

Each screen underwent individual property-by-property tuning:
- **Canvas scaling** configured for consistent rendering across PC, tablet, and mobile
- **Tween animations** on every interactive element (button hover states, panel slides, fade-ins)
- **Typography** consistent across screens using `Fondamento` typeface for thematic unity
- **Color system** derived from the in-game spell palette, creating continuity between gameplay and menus

### Key Screens

| Screen | Technical Highlights |
|---|---|
| **Spell Selection (Level Up)** | Full-screen overlay, blurred backdrop, spell cards animate in with elastic easing, rarity glow effects |
| **HUD (In-Run)** | Health bar, wave counter, spell icons with cooldown arc animations on a single Z-indexed `ScreenGui` |
| **Pause Menu** | Group vote UI with real-time approval percentage, non-blocking server state |
| **Artifact Picker** | Scrollable rarity-coded grid, tooltip hover system, selection confirmation animation |
| **Lobby Stats** | DataStore-driven stat cards with animated numerical counters |

### Responsive Layout System

| Form Factor | Adaptation |
|---|---|
| Desktop (16:9) | Full panels, expanded HUD |
| Tablet (4:3) | Scaled panels, touch-optimized button sizing |
| Mobile (9:16 portrait) | Collapsed HUD, thumb-zone-aware button placement |

---

## ⚙️ Development Pipeline & Production Notes

### Iteration History

```
Month 1 — Foundation
├── Enemy spawn system (GeneradorEnemigos) with time-based difficulty scaling
├── First spell: Disparo Mágico (server-side projectile prototype)
├── Core game loop: wave timer, kill counter, run state machine
└── Initial map layout: El Jardín (Fácil prototype)

Month 2 — Spell Architecture Overhaul
├── Client-Server authority split implemented across all spells
│   ├── Server: hitbox math, damage, state enforcement
│   └── Client: VFX, meshes, particle emitters
├── Fireball: sourced visual assets from Toolbox, fully re-architected into
│   server-authoritative movement system with client-side visual decoupling
└── Arcane Beam: first piercing hitscan implementation

Month 3 — Content Expansion
├── Artifact system (199+ items) with DataStore persistence
├── Legendary fusion system: Longinus, Gran Inundación, Genocidio
├── Boss encounter framework: Ritual Seal, Obelisk Chest, death sequence
├── La Ciudad Maldita and Las Backrooms maps
└── Multiplayer co-op: aggro redistribution, vote-pause, group chest logic

Ongoing — Polish, Balancing, Lobby Meta
├── UI/UX calibration (manual per-property tuning across all screens)
├── Lobby NPC dialogue trees and economy systems
├── Performance optimization: culling, LOD, adaptive particle budgets
└── Marketing: custom thumbnail campaign (art by pickle_remolacha)
```

### Asset Pipeline Strategy

A recurring challenge in Roblox development is the distinction between **raw asset acquisition** and **architectural integration**. *Wizard Survival* uses a deliberate pipeline:

1. **Source** — Meshes and particle systems acquired from Toolbox or authored in Studio as rapid prototyping tools
2. **Audit** — Each asset is analyzed for performance characteristics and behavioral hooks
3. **Re-architect** — Assets are fully decoupled from their original scripts and re-integrated into the game's server-authoritative architecture
4. **Validate** — Server-side hitbox math implemented independently using spatial algebra; visual asset and gameplay logic are separate concerns at all times

### Technology Stack

| Tool | Role |
|---|---|
| **Roblox Studio** | Development environment, scene authoring, in-editor testing |
| **Luau** | Server scripts, client controllers, module libraries, UI logic |
| **Roblox DataStore API** | Player persistence: currencies, unlocks, run statistics |
| **RemoteEvent / RemoteFunction** | Client ↔ Server communication boundary |
| **Blender 3D** | Asset creation, cinematic trailer (Python `bpy` scripting) |
| **Python** | External tooling: batch asset processing, automation scripts |
| **Git** | Version control |
| **AI-assisted workflows** | Algorithmic problem-solving, script acceleration, edge-case math validation — always under human review, testing, and refactoring |

---

## 📈 Launch Metrics, Credits & Roadmap

### Launch Performance

| Metric | Value |
|---|---|
| **Unique Visits** | 1,200+ at launch window |
| **Marketing Channel** | Custom thumbnail campaign — art by `pickle_remolacha` |
| **Leaderboard Presence** | Active placement in competitive global wave-count boards |
| **Platform** | Roblox (PC + Mobile) |
| **Retention Signal** | Organic return visits from players pursuing higher-difficulty completions |

### Credits

| Role | Credit |
|---|---|
| **Game Direction, Architecture & Level Design** | Game Director — months of continuous solo development and iteration |
| **Thumbnail Art, Miniatures & Visual Branding** | **pickle_remolacha** |

### Roadmap

#### Near-Term (v1.1 – v1.2)

- [ ] **New Map: El Abismo** — underground cave environment, vertical spawners
- [ ] **Expanded Boss Roster** — 3 new archetypes with unique telegraphed mechanics
- [ ] **Spell Evolution Tree UI** — visual grimoire showing full combination paths
- [ ] **4-Player Full Lobby** — expand from 2-player to 4-player squad co-op

#### Mid-Term (v2.0)

- [ ] **Raids Mode** — high-difficulty challenge runs designed for coordinated squads
- [ ] **Artifact Crafting** — combination system for upgrading artifact variants
- [ ] **Language Support** — EN / ES dual-language UI with regional selector

#### Long-Term Vision

- [ ] **Seasonal Events** — time-limited maps and exclusive artifact drops
- [ ] **Leaderboard Seasons** — competitive cycles with reset periods and exclusive rewards
- [ ] **Custom Difficulty Builder** — player-configurable modifier stack for self-imposed runs

---

<div align="center">

*Built from the ground up. Refined through iteration.*

**[▶ Play on Roblox](#) · [💬 Discord](#) · [📋 Roadmap](#)**

</div>
