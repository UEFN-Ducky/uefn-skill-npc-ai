---
name: npc-ai
description: "UEFN NPC AI exact recipes — character definitions, equipment/weapons, Verse behaviors, SG projectiles, damage, shoot-down, spawning. Recreate Roguelike enemies 1:1."
license: Ducky Source-Available License v1.0
metadata:
  label: "UEFN NPC AI & Enemies"
  version: 6
  author: Iliya Kovachki
  copyright: Copyright 2026 Iliya Kovachki
  allow_redistribute: false
  managed_by: uefn-ducky
  source: store
  store_slug: npc-ai
---

# NPC AI & Enemies — recreate index



Source project: `Roguelike` — content under `/AI/`, Verse under `Verse/GameDevices/AISystems/`.



**Stay inside the active UEFN project folder** when creating meshes, materials, NPCDefs, or Verse files.



Generic pattern primers (keep linked, do not duplicate):



- `skill_read_subskill("verse", "sys_npc_ai")` — generic `npc_behavior` skeleton

- `skill_read_subskill("animation", "npc_characters")` — physics → AnimPreset → BP → NPCDef → spawner **via tools** (never Details)
- `skill_read_subskill("animation", "npc_ecosystem")` — multi-species session registries, play-dead, hardcoded same-module clips

- `skill_read_subskill("animation", "npc_items")` — socket attach MCP cookbook (hats/weapons)

- `skill_read_subskill("animation", "retargeting")` — IK Rig / Biped chains



This pack holds the **exact Roguelike recipes** agents need to recreate the same enemies, projectiles, and hurt logic.



## Hard rules (read first)



| Trap | Rule |

|------|------|

| `npc_behavior` calling `GetPlayspace()` | **Fails.** Use `Behavior.GetEntity[]` → `GetPlayspaceForEntity[]` → `GetPlayers()`. |

| Behaviors calling `fort_character.Damage` | **Never.** Only `EnemyApplyHitDamage` / `EnemyMeleeHit` / `EnemyAOEHit` / `EnemyFireSGProjectile` in `enemy_combat_helpers.verse`. |

| Creative `SpawnProp` arrows vs walls | Invisible to `FindSweepHits`. Use Scene Graph spheres + `keyframed_movement_component`. Walls must be Queryable SG entities. |

| Fortnite guns vs SG projectiles | Guns cannot damage SG spheres. Use `enemy_projectile_shootdown` (Fire held + aim ray). |

| Mesh facing wrong way | Fix character asset axis — do **not** reintroduce global Focus yaw hacks. |

| Held sword = damage | **No.** Weapon mesh is cosmetic; hits go through combat helpers. See `equipment`. |



## Subskill map



| Want | Load |

|------|------|

| Enemy roster (NPCDef ↔ BP ↔ mesh ↔ mat ↔ behavior) | `roster` |

| Build / wire `NPCCharacterDefinition` | `character_definitions` |

| Swords / bows / custom weapons on NPCs | `equipment` |

| Helpers + default melee loop | `behavior_base` |

| Melee / charger / sploder / hooker / boss | `behaviors_melee` |

| Archer / mage / grenadier | `behaviors_ranged` |

| How projectiles spawn + hurt the player | `projectiles` |

| How players shoot projectiles down | `shootdown` |

| Spawn manager / dungeon / SG walls / lane test | `spawning` |



## Folder layout (create under active project)



```

Content/AI/

  NPCDef_<Name>.uasset

  BP_<Name>.uasset          # optional editor preview

  AnimPresets/AP_*_Locomotion.uasset

  EnemyMeshes/SK_*(_Real).uasset

  EnemyMats/MI_*_Own.uasset

Content/Verse/GameDevices/AISystems/

  enemy_ai_helpers.verse

  enemy_combat_helpers.verse

  enemy_*_behavior.verse

  enemy_spawn_manager.verse

  enemy_projectile_shootdown.verse

  sg_collision_helpers.verse

  dungeon_level_spawner.verse   # optional

```



## New-enemy checklist



1. Mesh + physics asset + restored anims → AnimPreset (`npc_characters`).

2. Weapon look: baked mesh **or** socket-attach custom prop (`equipment`).

3. `NPCCharacterDefinition` + Health + VerseBehavior → pick behavior class (`character_definitions`, `roster`).

4. Fill `@editable` AttackAnim / ranges on the definition.

5. Place `npc_spawner_device` → wire into `enemy_spawn_manager` (`spawning`).

6. If ranged: place `enemy_projectile_shootdown` + SG Queryable walls (`projectiles`, `shootdown`).

7. `workspace_list_verse_errors` → PIE chase / visible weapon / hit prints / elim count.
