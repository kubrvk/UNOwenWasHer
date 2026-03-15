# U.N. Owen Was Her

> **Third-person action horror** — Unreal Engine 5.3 · C++ · Solo Development  
> [Steam Page](https://store.steampowered.com/app/3420540/UN_Owen_Was_Her) · [ArtStation](https://www.artstation.com/kubrik)

![image](https://shared.akamai.steamstatic.com/store_item_assets/steam/apps/3420540/000f0c55e13d6080aaed238df1661a46c4079ada/logo.png?t=1767454465)

---

## Overview

U.N. Owen Was Her is a third-person atmospheric horror game built in Unreal Engine 5.3 using C++. The player is trapped inside the Scarlet Mansion — a haunted estate populated by autonomous entities derived from the U.N. Owen figure. The game is structured around a **Feed-or-Fight** decision loop: enemies can be pacified through a hunger-state system that transforms their behavior and form, or engaged directly using a limited-ammo weapon set. Puzzle progression is tied to entity state, environmental manipulation, and resource management.

All gameplay systems, enemy AI, characters, shaders, and tooling were developed by a single developer.

---

![image](https://shared.akamai.steamstatic.com/store_item_assets/steam/apps/3420540/extras/4ceaac1e6bb275e6f2a31a97f678462c.avif?t=1767454465)


## Engine & Technical Stack

| Layer | Technology |
|---|---|
| Engine | Unreal Engine 5.3 |
| Primary Language | C++ (gameplay, AI, systems) |
| Scripting / Prototyping | Unreal Blueprint (UI scaffolding, cutscene sequencing) |
| Rendering | Lumen (dynamic GI/shadows), custom post-process for horror atmosphere |
| AI | Unreal Behavior Tree + custom `UBTTask`/`UBTDecorator` nodes |
| Physics | Chaos — used for thrown food projectiles, physics-driven puzzle props |
| Save System | `USaveGame` + fireplace checkpoint architecture |
| Platform | PC (Win64/Linux), Steam SDK |
| 3D Pipeline | ZBrush → Maya → Substance Painter → UE5 |
| Shader Authoring | UE Material Editor + HLSL custom nodes |

---

## Architecture Overview

```
UNOwen/
├── Source/
│   ├── Core/
│   │   ├── UNOwenCharacter.h/.cpp             # Player character
│   │   ├── UNOwenGameMode.h/.cpp              # Session, progression gating
│   │   └── UNOwenGameState.h/.cpp             # Global state: seals, entity registry
│   ├── Systems/
│   │   ├── HungerSystem/                       # Entity hunger state, feeding, transformation
│   │   ├── CombatSystem/                       # Weapon management, ammo economy, hit resolution
│   │   ├── AIController/                       # Entity patrol, detection, state machines
│   │   ├── PuzzleSystem/                       # Light puzzles, gear puzzles, seal logic
│   │   ├── FoodSystem/                         # Food item types, throw mechanics, lure AI
│   │   ├── SealSystem/                         # Dungeon seal progression, unlock sequencing
│   │   ├── SaveSystem/                         # Fireplace checkpoint, inventory persistence
│   │   └── TransformationSystem/               # Entity form change on full feed
│   ├── Entities/
│   │   ├── UNOwenBase.h/.cpp                  # Base entity class
│   │   ├── UNOwenRoaming.h/.cpp               # Standard roaming variant
│   │   └── UNOwenOriginal.h/.cpp              # Final boss variant
│   ├── Environment/
│   │   ├── MansionFrontWing/                   # Halls, bedrooms, fireplace area
│   │   ├── Library/                            # Multi-floor library zone
│   │   └── UndergroundDungeon/                 # Sealed dungeon, final area
│   └── UI/
│       ├── HUD/                                # Hunger indicator, ammo counter, item slots
│       └── Menus/                              # Storage access, pause, inventory
```

---

## Core Systems: Technical Detail
![image](https://shared.akamai.steamstatic.com/store_item_assets/steam/apps/3420540/extras/4aab3ddea795980257c11b0a92cffc84.avif?t=1767454465)

### 1. Hunger & Transformation System

The Hunger System is the central design mechanic of U.N. Owen Was Her. Every entity (`AUNOwenBase`) maintains a `FHungerState` struct tracking `CurrentHunger`, `MaxHunger`, and `TransformationThreshold`.

**Hunger State Machine:**

```
HUNGRY ──feed──► SATIATED ──threshold reached──► TRANSFORMED (Human Form)
   │                                                      │
   └─── attacked / time decay ──────────────────► HOSTILE (resets hunger)
```

- `CurrentHunger` increases on each successful feeding interaction.
- On reaching `TransformationThreshold` (entity-specific, defined per `UEntityDataAsset`):
  - Entity plays transformation animation sequence.
  - AI Behavior Tree switches from `BTT_HostilePatrol` to `BTT_HumanAssist` subtree.
  - Entity collision profile changes: no longer damages player on overlap.
  - Entity becomes available as a **puzzle assistant** (see Puzzle System).
- Hunger decays over time via a `FTimerHandle`-driven tick — transformed entities revert if left unattended for a configurable duration (designer-tunable per entity type).
- Attacking a hungry or satiated entity resets hunger to 0 and triggers `BTT_HostileAlert`.

```cpp
void AUNOwenBase::ApplyFeeding(float NutritionValue)
{
    HungerState.CurrentHunger = FMath::Clamp(
        HungerState.CurrentHunger + NutritionValue,
        0.f,
        HungerState.MaxHunger
    );

    if (HungerState.CurrentHunger >= HungerState.TransformationThreshold)
    {
        TriggerTransformation();
    }
    else
    {
        SetAIState(EEntityAIState::Satiated);
    }
}
```

---

### 2. Food System

Food items are defined via `UFoodItemDataAsset`, specifying `NutritionValue`, `LureRadius`, `ThrowArc`, and `ItemMesh`.

**Three interaction modes:**

| Mode | Mechanic | Implementation |
|---|---|---|
| **Direct Feed** | Player interacts with entity at close range; hunger applies immediately | `UInteractionComponent::OnFeedEntity()` |
| **Throw Lure** | Food thrown as physics projectile; on landing, broadcasts lure event in radius | `AFoodProjectile` + `UGameplayStatics::ApplyRadialDamage`-equivalent custom broadcast |
| **Inventory Hold** | Food retained for later use; conserves ammo by avoiding combat | `UInventoryComponent` slot management |

**Lure AI Response:**
- `AUNOwenAI` subscribes to `OnFoodLanded` delegate via `UHungerSubsystem`.
- Entities within `LureRadius` receive a `MoveToLocation` BT task override — interrupts current patrol and pathfinds to food location.
- Multiple entities can be lured simultaneously; food item is consumed on first entity arrival.

**Food as Resource Economy:**
- Food items are finite and distributed through the mansion via `AFoodPickupActor` placements.
- Designer-placed food scarcity creates tension between luring/feeding and ammo conservation.
- Cakes, pastries, and sweets have differentiated `NutritionValue` and `LureRadius` values — heavy pastries lure at short range with high nutrition; light sweets lure broadly with low nutrition.

---

### 3. Combat System

Combat is deliberately constrained — ammunition is a scarce resource, intended to be a last resort rather than a primary strategy.

**Weapon Architecture:**
- Weapons defined via `UWeaponDataAsset`: fire rate, damage, ammo type, reload time, projectile class.
- `UWeaponComponent` manages active weapon, ammo pool per type, and reload state.
- No ammo respawn — all ammunition is world-placed. Total ammo budget is fixed per playthrough.

**Weapon Categories:**

| Category | Role | Design Intent |
|---|---|---|
| Firearms (pistol, shotgun) | Mid-range; standard ammo | General deterrent; limited but accessible |
| Heavy Launchers | Area denial, stagger | Reserved for boss encounters or emergencies |

**Hit Resolution:**
- Hitscan weapons use line trace (`ECC_GameTraceChannel_Weapon`) with material-based surface response.
- Projectile weapons use `AUNOwenProjectile` with `UProjectileMovementComponent`.
- Hit applies damage via `UGameplayStatics::ApplyDamage`; entity `TakeDamage` override routes to `HungerSystem::OnEntityDamaged` to reset hunger.

**Ammo Economy Design:**
- Total ammo across the mansion is intentionally insufficient to neutralize all entities by combat alone.
- Players who exhaust ammo early must rely exclusively on food mechanics — this is an intentional design pressure, not a failure state.

---

### 4. Entity AI System
![image](https://shared.akamai.steamstatic.com/store_item_assets/steam/apps/3420540/extras/25a69bfbc4efe1a46e81fabedcc24c33.avif?t=1767454465)<br>
All entities are `AUNOwenBase` subclasses controlled by `AUNOwenAIController`, using Unreal's Behavior Tree with custom C++ task and decorator nodes.

**AI State Architecture:**

```
EEntityAIState:
  ├── Patrol          → Roam waypoint path, periodic idle
  ├── Investigating   → Move to last known stimulus location
  ├── Hostile         → Chase and attack player
  ├── Satiated        → Slow patrol, reduced detection range
  ├── Lured           → Override path to food location
  └── HumanAssist     → Follow transformed entity behavior, idle near puzzles
```

**Perception:**
- `UAIPerceptionComponent` with sight and hearing channels.
- Sight: cone-based, range and angle tuned per entity variant.
- Hearing: food throws and player movement above speed threshold generate noise events.
- Transformed (human-form) entities have perception disabled — they no longer register player as threat.

**Patrol System:**
- Waypoints are `APatrolPoint` actors placed in editor, referenced via `TArray<APatrolPoint*>` on each entity.
- Patrol order: sequential or random (per-entity config in data asset).
- Wait duration at each waypoint defined per `FPatrolConfig`.

**Detection Escalation:**
- Entity detects player → enters `Investigating`.
- Direct line-of-sight confirmed → escalates to `Hostile`.
- Loses sight for > N seconds → returns to patrol from last known position.
- Hearing a food throw → enters `Lured` (overrides `Investigating`/`Hostile`).

---

### 5. Puzzle System

Puzzle progression gates mansion exploration. Two primary puzzle types are implemented, with entity-assistance as an optional accelerant.

**Light Puzzles:**
- Require directing light beams via rotatable mirror props (`ARotatableMirror`).
- Mirror rotation driven by player interaction; rotation snaps to 45° increments using `FRotator` quantization.
- Light beam implemented as a `USpotLightComponent` with per-frame trace; on hitting a `AReceptorActor`, triggers the associated event.
- Some receptors require simultaneous illumination — `AReceptorActor` broadcasts to a `UPuzzleCoordinatorComponent` that tracks how many receptors are active.

**Gear Puzzles:**
- Interconnected gear meshes must be activated in the correct sequence or combination.
- Each `AGearActor` has a `FGearState` struct: `bIsEngaged`, `DependencyList` (other gears that must be active first).
- `UPuzzleCoordinatorComponent` evaluates the full dependency graph on any state change; triggers `OnPuzzleSolved` delegate when all conditions met.

**Entity Assistance:**
- Transformed (human-form) U.N. Owen entities can interact with specific puzzle props.
- `AUNOwenBase::bCanAssistPuzzles` flag enabled post-transformation.
- Player can direct an assisting entity to a puzzle actor via an interaction prompt — entity pathfinds to position and executes a `UBTTask_ActivatePuzzleProp` task.
- This reduces puzzles from requiring exact player positioning to allowing remote activation — necessary in areas where the player cannot safely stand.

---

### 6. Seal System & Dungeon Progression

The mansion is gated by a seal-based progression structure. Seals are `ASealActor` instances managed by `USealSubsystem`.

**Seal Architecture:**
- Each seal has: required trigger conditions (puzzle solved, entity count fed, item acquired), a locked door/passage reference, and a state (`Locked` / `Breaking` / `Broken`).
- `USealSubsystem` maintains a registry of all seals. On condition fulfillment, `OnSealConditionMet` broadcasts and the subsystem evaluates whether all conditions for that seal are satisfied.
- Breaking a seal plays a destruction sequence (Sequencer-driven) then enables passage to the next zone.

**Progression Zones:**

```
Mansion Front Wing (Halls, Bedrooms)
        │
        ▼ [Seal 1: Library Key Puzzle]
Multi-Floor Library
        │
        ▼ [Seal 2: All Wing Entities Fed or Neutralized]
Hidden Underground Areas
        │
        ▼ [Final Seal: Original U.N. Owen Dungeon Gate]
Underground Dungeon ──► Final Boss
```

- The final seal requires breaking a sequence of sub-seals distributed across all prior zones, ensuring full exploration before dungeon access.
- `UNOwenGameState` tracks global seal status, replicated for potential future co-op support.

---

### 7. Boss System — The Original U.N. Owen
![image](https://shared.akamai.steamstatic.com/store_item_assets/steam/apps/3420540/eaf3d1ac4c086749b33fbb756326ecc647dd88f6/ss_eaf3d1ac4c086749b33fbb756326ecc647dd88f6.800x600.jpg)<br>
The final boss (`AUNOwenOriginal`) is a distinct subclass of `AUNOwenBase` with an extended phase-based behavior tree and a bullet-hell attack pattern system.

**Phase Architecture:**
- 3 combat phases, gated by health percentage thresholds.
- Phase transitions trigger via `OnPhaseChange` delegate; new Behavior Tree subtree is injected dynamically using `UBehaviorTreeComponent::SetDynamicSubtree`.
- Each phase introduces new attack patterns and movement speeds.

**Bullet-Hell Pattern System:**
- Attack patterns defined in `FBulletPatternConfig` data assets: projectile count, arc spread, rotation speed, fire rate, projectile type.
- `ABulletPatternActor` is spawned by the boss AI during ranged attack phases; manages its own projectile spawning loop via `FTimerHandle`.
- Projectile types: standard damage, lure-type (draws player toward boss), and time-delayed burst.
- Pattern difficulty scales per phase: phase 1 uses simple radial spreads; phase 3 introduces interleaved multi-origin patterns.

**Hunger Interaction — Final Boss:**
- The original U.N. Owen is not fully transformable but *can* be partially pacified.
- Feeding during combat temporarily suppresses one attack phase: boss enters `Satiated` for a configurable window, reducing aggression.
- This creates a resource decision: spend food to create safe windows, or conserve food and rely on combat.
- Full defeat requires reaching 0 HP — feeding alone cannot end the encounter.

---
![image](https://shared.akamai.steamstatic.com/store_item_assets/steam/apps/3420540/97a04d66ce55ffadcf87a570af48e5e08b1ecf1d/ss_97a04d66ce55ffadcf87a570af48e5e08b1ecf1d.800x600.jpg)

### 8. Save System — Fireplace Checkpoint

Progress is saved exclusively at fireplace objects (`AFireplaceActor`) distributed sparingly through the mansion.

**Checkpoint Architecture:**
- `AFireplaceActor` triggers a `USaveManager::SaveCheckpoint()` call on player interaction.
- Saved data: player position (nearest fireplace index, not raw coordinates), inventory state, ammo counts, entity hunger states, puzzle solved flags, seal states, and active zone.
- Entity states are serialized as `TArray<FEntitySaveData>` — each entry stores entity unique ID, current hunger, and transformation status. On load, entities are restored to their saved state rather than respawning fresh.
- Async save via `UGameplayStatics::AsyncSaveGameToSlot` to avoid frame hitching.

**Storage Access:**
- Fireplace interaction also opens `UStorageWidget` — a stash shared across all fireplace locations.
- Storage persists across saves; items placed in storage are always available at any fireplace.
- Implemented as a separate `FStorageSaveData` block within the save file, loaded independently.

**Design Intent:**
- Fireplace scarcity makes save timing a conscious decision — reinforces horror pacing and tension between progress preservation and resource use.
- No autosave outside of fireplace interaction.

---

### 9. Environmental Design & Zones
![image](https://shared.akamai.steamstatic.com/store_item_assets/steam/apps/3420540/18fd9f0ad2d8f2871d1db85bb763762cd21afdb5/ss_18fd9f0ad2d8f2871d1db85bb763762cd21afdb5.800x600.jpg)<br>
**Mansion Front Wing:**
- Interconnected halls and bedrooms with high entity density early in the game.
- Fireplace is located here — functions as the primary early-game safe zone.
- Props and interactive objects are `AInteractableActor` subclasses with `UInteractionComponent`.

**Multi-Floor Library:**
- Vertical multi-level space; multiple floors connected by staircases and catwalks.
- Higher entity variety and density; food scarcity increases.
- Contains the majority of gear puzzles.
- Lumen GI used for dramatic contrast between lit reading areas and shadowed stacks.

**Underground Dungeon:**
- Accessible only after all seals broken.
- Linear zone leading to final boss arena.
- Environmental hazards implemented as `ATriggerVolume` actors with damage-over-time components.
- Final boss arena: circular space with cover objects as destructible `ADestructibleActor` props.

**Lighting Architecture:**
- All dynamic lighting via Lumen; no baked lightmaps.
- Each zone uses a `APostProcessVolume` with zone-specific color grading, vignette, and chromatic aberration settings.
- Fireplace uses a dynamic point light with flickering driven by a `UCurveFloat` on light intensity — no Blueprint logic, pure C++ timeline component.

---

### 10. Atmosphere & Visual Systems

**Post-Process Horror Pipeline:**
- Screen-space lens distortion on entity proximity: scaled via distance to nearest `AUNOwenBase` actor, updated per-frame.
- Desaturation increases as player health decreases — material scalar parameter driven by `UHealthComponent`.
- Custom depth pass used for entity silhouette rendering through walls when in `Investigating` state (proximity warning system).

**Dynamic Lighting:**
- Entities carry dynamic point lights that shift color based on hunger state (cool blue → warm amber → white on transformation).
- Light color and intensity are driven by `FHungerState.CurrentHunger` mapped to a `UCurveLinearColor` asset.

**Particle & VFX:**
- Transformation sequence uses Niagara particle system triggered from C++ via `UNiagaraFunctionLibrary::SpawnSystemAtLocation`.
- Food throw impact: Niagara splash effect at landing location, also serves as visual cue for lure radius.

---

## Performance Targets & Optimization

| Target | Approach |
|---|---|
| 60 fps (PC, 1080p+) | LOD chains on all entity meshes; Nanite on static environment geo |
| AI performance | Behavior Tree updates throttled for off-screen entities; perception disabled for transformed entities |
| Memory | Async asset loading for zone transitions; dungeon zone loaded async on final seal break |
| Lighting | Lumen hardware ray tracing on supported GPUs; software fallback for mid-range hardware |
| Tick budget | Entity hunger decay via `FTimerHandle` (not per-frame tick); only `Hostile` AI state entities tick at full rate |
| Physics | Chaos physics only on thrown food projectiles and destructible props; no persistent physics simulation on entities |

---

## Development Scope

| Category | Detail |
|---|---|
| Developer count | 1 (solo) |
| Engine | Unreal Engine 5.3 |
| Languages | C++, HLSL (custom shader nodes) |
| 3D Assets | All original — modeled, textured, rigged, animated by developer |
| Zones | 3 major (Front Wing, Library, Underground Dungeon) |
| Entity variants | Multiple U.N. Owen variants + Original final boss |
| Gameplay systems | 10+ discrete systems (see above) |
| Platform | PC Windows / Linux (Steam) |
| Development tools | UE5 Editor, ZBrush, Maya, Blender, Substance Painter, Photoshop, After Effects |

---

## Related Projects

| Project | Description |
|---|---|
| [TIME SOUL](https://store.steampowered.com/app/2928270/TIME_SOUL) | Souls-like action platformer; parkour, time-as-resource, procedural gen — UE5.1 |
| [Olympus of the Heavens](https://store.steampowered.com/app/3358020/Olympus_of_the_Heavens) | Isometric co-op ARPG; procedural gen, Steam networking — UE5 |
| [Blood Garden](https://kubrik.itch.io/bloodgarden) | Souls-like melee combat; stamina system, parry, enemy AI |
| [Royal Jump](https://play.google.com/store/apps/details?id=com.Kubrick.RoyalJump) | Mobile platformer; touch controls, physics movement |
| [ArtStation Portfolio](https://www.artstation.com/kubrik) | 3D modeling — characters, creatures, props, environments |

---

## Developer

**Kubrick** — Developer & 3D Artist  

[Steam](https://store.steampowered.com/search/?developer=Kubrik) · [ArtStation](https://www.artstation.com/kubrik) · [itch.io](https://kubrik.itch.io)

---
![image](https://shared.akamai.steamstatic.com/store_item_assets/steam/apps/3420540/5cc027cc74f9a91724e1d4d0098e951c3fa204d5/library_hero.jpg?t=1767454465)

*All code, design, and custom assets produced by a single developer. No third-party gameplay code used in core systems.*
