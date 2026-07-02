# King's Stand — Game Design Spec

**Date:** 2026-07-02
**Status:** Approved by Sindre (concept, roadmap, and architecture sections) — pending final spec review
**Repo:** https://github.com/sindresorensen02-blip/IOSGAME
**Working title:** "King's Stand" (must not ship as "Kingshot" — trademark of Century Games; rename is a find-and-replace away)

## 1. Concept

An endless-survival action roguelite for iOS, remaking the Kingshot playable-ad mini-game as a complete game. The player is the King, alone inside a palisade fort in the desert. Waves of enemies march down the road and attack the walls. The player steers the King with one thumb; he auto-fires his crossbow. Dead enemies drop scrap. The player hauls scrap to a trader counter where customers queue to buy it, collects the resulting coin piles, and converts profit into towers and upgrades during the run. When the walls fall and the King dies, the run ends — earnings convert to Crowns, spent on permanent Dynasty upgrades that strengthen every future run.

Reference material: three screenshots of the original playable ad (isometric low-poly fort, knights marching a sand road, crossbow trajectory line, dashed 10-coin build plots, trader counter with customer queue, coin pallet, virtual joystick, coin + scrap HUD counters).

### Design pillars

1. **One-thumb play** — joystick only. Auto-aim, auto-sell at the counter, auto-collect by walking over pickups. The single exception: tap anywhere to trigger the hero ability (M2).
2. **Greed vs. safety** — scrap and coins sit where enemies are heading. Every pickup is a risk/reward decision.
3. **Death is progress** — every run funds the Dynasty tree; the next run always starts stronger.

### The 60-second loop

Enemies path toward gate → King auto-fires → enemies drop scrap → backpack fills (limited capacity) → drop scrap at trader counter → customer queue buys it, coins pile on the pallet → walk over coins to collect → spend on build plots (towers) and in-run upgrades → bigger waves arrive → repeat until breached → death → run summary → Crowns → Dynasty tree → next run.

### Failure condition

Enemies breach the gate/walls (walls and gate have HP), then hunt the King directly. King HP reaches zero → run over.

## 2. Milestone roadmap

Build approach: **milestone ladder** — four shippable milestones, each a complete playable game strictly better than the last. Each milestone gets its own implementation plan; this spec is the source of truth for all four.

### M1 — Core loop (playable v0.1)

- One desert fort map matching the ad layout: palisade walls, gate, sand road, trader counter at the fence, coin pallet, 4 build plots.
- King: floating virtual joystick movement (WASD fallback on desktop), auto-aim crossbow (nearest target priority), HP bar.
- 3 enemy types: **Knight** (baseline melee, attacks gate), **Bandit** (fast, low HP), **Brute** (armored, slow, high gate damage).
- Endless wave director: waves spawn on a timer with escalating count/composition/HP multipliers; no win state.
- Economy chain: scrap drops → backpack (capacity-limited) → trader counter → customer queue buys at a fixed rate → coin piles on pallet → walk-over collect.
- 2 tower types on dashed build plots: **Crossbow tower** (single target), **Ballista** (piercing line shot, slower).
- Gate + wall segments with HP; breach opens a path for enemies to enter and target the King.
- Death → run summary (waves survived, kills, coins earned) → Crowns awarded (initial formula, tuned during balancing: `crowns = waves_survived * 2 + coins_earned / 10`, rounded down) → Dynasty tree.
- Dynasty tree v1, 8 permanent upgrades: crossbow damage, backpack size, wall/gate HP, starting coins, tower cost discount, trader price bonus, unlock 5th/6th build plot, King move speed.
- Screens: main menu, HUD, pause, run summary, Dynasty tree. Settings: sound on/off.
- Save: Crowns, Dynasty purchases, lifetime stats persist. Runs are ephemeral.
- Runs on Windows desktop for development testing.

### M2 — Combat depth

- 3 additional hero weapons unlocked via Dynasty: multi-shot bow, AoE hammer, magic staff. Weapon selected before a run.
- Enemy roster grows to 8+: shieldbearer (blocks frontal shots), archer (ranged attacks on King/towers), horseman (fast, heavy gate damage), sapper (bombs walls), elite variants of base types.
- Mini-boss every ~5 waves; boss every ~15 waves (Siege Giant, Warlord). Bosses drop relics (M3 currency for the Relic Collector).
- Hero active ability with cooldown, triggered by tapping anywhere (e.g. rally stomp knockback).
- In-run tower upgrading: pay coins to tier a placed tower up (3 tiers per type).

### M3 — Economy depth

- Multiple traders with distinct demand and prices: **Scrap Trader** (base), **Armorsmith** (rare armor drops), **Relic Collector** (boss relics).
- Random events: merchant caravan (limited-time bulk-buy price bonus), night raid (double spawns, double drops), price surge (one good pays 2x briefly).
- New buildings on plots: wall-repair station, farm (passive coin trickle), war banner (buff aura for towers/King).
- Porter NPC (Dynasty unlock): auto-hauls scrap from the field to the counter.

### M4 — Polish & platform

- Cosmetics purchasable with Crowns: King skins, cape colors, crossbow skins. No IAP in v1.
- Game Center: leaderboards (highest wave, longest survival time) and achievements.
- Sound design: music loop, SFX pass (shots, hits, sales, coins), iOS haptics.
- iOS export: icons, launch screen, TestFlight distribution, App Store assets, 60 fps performance pass (draw call budget, mobile renderer tuning).

## 3. Technical architecture

**Stack:** Godot 4.x (latest stable), GDScript, true 3D with a fixed isometric-angle camera matching the ad. Mobile renderer. Developed on Windows; iOS export in M4.

### Repo layout

```
IOSGAME/
├── project.godot          # Godot project root
├── autoload/              # global singletons
│   ├── game_state.gd      #   current run state (coins, scrap, wave)
│   ├── meta_state.gd      #   Crowns + Dynasty tree (persisted)
│   ├── save_manager.gd    #   JSON save to user://, versioned
│   └── event_bus.gd       #   decoupled signals (enemy_died, scrap_sold, wall_breached…)
├── scenes/
│   ├── main_menu/  run/  run_summary/  dynasty/
├── entities/
│   ├── king/  enemies/  towers/  traders/  pickups/  fort/
├── systems/
│   ├── wave_director.gd   # spawn curves, escalation
│   ├── economy.gd         # prices, drops, customer queue
│   └── build_system.gd    # plots, tower placement
├── ui/                    # HUD, joystick, menus (Godot themes)
├── assets/                # Kenney/Quaternius GLTF imports (CC0)
├── tests/                 # GUT unit tests for systems logic
└── docs/superpowers/specs/  # design docs (this file)
```

### Component boundaries

- **Every entity is its own scene** (king, each enemy type, each tower, trader, pickup). Entities never reference each other directly; they emit and listen on `event_bus` signals. This keeps M2/M3 additions from touching M1 code.
- **Systems are pure-logic GDScript** (wave math, economy pricing, build rules) with no scene-tree dependencies, so they are unit-testable headlessly.
- **Autoloads hold state**: `game_state` (per-run, reset on run start), `meta_state` (persistent), `save_manager` (serialization only), `event_bus` (signals only, no state).

### Data flow

Input (joystick/WASD) → King controller → combat events on `event_bus` → `game_state` mutations (scrap, coins) → HUD reacts via signals. On death: `game_state` snapshot → run summary → Crowns formula → `meta_state` → `save_manager.save()`.

### Save system & error handling

- JSON at `user://save.json` with a `schema_version` field. Loads run through per-version migration; unknown/corrupt saves are backed up to `user://save.bak` and a fresh save is created (never crash on load).
- Only meta persists: Crowns, Dynasty purchases, cosmetics, lifetime stats, settings.
- All economy math clamps at zero (no negative coins/scrap); build actions validate cost server-side of the click (system rechecks, UI never trusted).

### Testing strategy

- **GUT** (Godot Unit Test) for systems: wave escalation curves, economy pricing, Crowns formula, Dynasty effects, save migration. Balance-critical logic is developed test-first.
- Entities/visuals are validated by playing the Windows build; no unit tests for rendering or feel.
- M4 adds on-device TestFlight testing.

### Input

Floating virtual touch joystick (appears where the thumb lands, like the ad). Desktop fallback: WASD + mouse for development. M2's ability trigger: any tap/click that is not the joystick.

## 4. Art & design pipeline

- **3D world art:** Kenney and Quaternius CC0 low-poly packs (medieval knights, villagers, forts, props) imported as GLTF, recolored in-engine to the game palette. Zero cost, no licensing risk, matches the ad's aesthetic.
- **Palette:** desert sand exterior, fort-green interior ground, warm wood palisade, crown-gold accents (mirrors the ad).
- **UI/design system:** a claude.ai/design design-system project ("King's Stand DS") created and synced via DesignSync. Contents: palette tokens, typography, HUD layout, buttons, panels, run-summary and Dynasty-tree screen designs as preview components. Godot's UI theme mirrors those tokens; restyles in Claude Design are synced back into the Godot theme by Claude Code. Sync is incremental and one component at a time.

## 5. Risks & constraints

- **iOS signing requires a Mac or a cloud build/signing service** — not needed until M4, but budget for it.
- **Trademark:** ship name must differ from "Kingshot"; art must be original/CC0, not ripped from the ad.
- **Godot iOS export** is mature but less battle-tested than Unity's; M4 reserves time for export debugging.
- **Scope:** full-fat (M1–M4) is a multi-month project. The milestone ladder means a playable game exists from M1 onward.

## 6. Out of scope (v1)

- In-app purchases, ads, or any monetization.
- Multiplayer or cloud saves.
- Android (possible later — Godot exports it trivially, but v1 targets iOS).
- Localization beyond English (Norwegian can be added in M4 if desired).

## 7. Implementation order

M1 is specced above in full and goes to an implementation plan next (writing-plans). M2–M4 each get their own plan when their turn comes, using this spec as the source of truth.
