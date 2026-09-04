---
metadata:
  origin: store
---

# Enemy roster (Roguelike `/AI/`)

Exact content map. Paths are under the Roguelike project Content root unless noted.

Source marketplace pack meshes/anims live in `CityofBrass_Enemies/` (Meshes/Enemy/*, Animations_Restored/*). Project-owned copies / retargets sit in `AI/EnemyMeshes`, `AI/EnemyMats`, `AI/AnimPresets`.

## Roster table

| Enemy | NPCDef | Preview BP | Mesh (project / source) | Own MI | AnimPreset | Verse behavior |
|-------|--------|------------|-------------------------|--------|------------|----------------|
| Archer | `NPCDef_Archer` | `BP_Archer` | `CityofBrass_Enemies/Meshes/Enemy/Archer/Archer` | `MI_Archer_Own` | (retarget to corpse/locomotion family) | `enemy_archer_behavior` |
| Charger | `NPCDef_Charger` | `BP_Charger` | `CityofBrass_Enemies/.../Charger` | `MI_Charger_Own` | `AP_Charger_Locomotion` | `enemy_charger_behavior` |
| Corpse Basic | `NPCDef_CorpseBasic` | `BP_CorpseBasic` | `AI/EnemyMeshes/SK_CorpseBasic(_Real)` | — | `AP_Corpse_Locomotion` | `roguelike_enemy_behavior` or `enemy_melee_behavior` |
| Corpse Melee | `NPCDef_CorpseMelee` | `BP_CorpseMelee` | `SK_CorpseMelee(_Real)` | `MI_CorpseMelee_Body_Own`, `MI_CorpseMelee_Clothes_Own` | `AP_Corpse_Locomotion` | `enemy_melee_behavior` |
| Corpse Spear | `NPCDef_CorpseSpear` | `BP_CorpseSpear` | `SK_CorpseSpear(_Real)` | `MI_CorpseSpear_Body_Own`, `MI_CorpseSpear_Own` | `AP_Corpse_Locomotion` | `enemy_spear_behavior` |
| Corpse Sword | `NPCDef_CorpseSword` | `BP_CorpseSword` | `CityofBrass_Enemies/.../Sword` or corpse sword mesh | `MI_CorpseSword_Armor_Own`, `MI_CorpseSword_Body_Own` | `AP_Corpse_Locomotion` | `enemy_sword_behavior` |
| Grenadier | `NPCDef_Grenadier` | `BP_Grenadier` | `SK_Grenadier(_Real)` | `MI_Grenadier_Body_Own`, `MI_Grenadier_Clothes_Own`, `MI_Grenadier_Hat_Own` | — | `enemy_grenadier_behavior` |
| Hooker | `NPCDef_Hooker` | `BP_Hooker` | `SK_Hooker(_Real)` | `MI_Hooker_Own` | — | `enemy_hooker_behavior` |
| Hooker Boss | `NPCDef_HookerBoss` | `BP_HookerBoss` | (same family / larger) | `MI_HookerBoss_Own` | — | `enemy_boss_behavior` |
| Large | `NPCDef_Large` | `BP_Large` | `SK_Large(_Real)` | — | — | `enemy_melee_behavior` (or boss) |
| Large Charger | `NPCDef_LargeCharger` | `BP_LargeCharger` | `SK_LargeCharger(_Real)` | `MI_LargeCharger_Own` | — | `enemy_large_charger_behavior` |
| Large Sword | `NPCDef_LargeSword` | `BP_LargeSword` | `SK_LargeSword(_Real)` | — | — | `enemy_sword_behavior` |
| Sploder | `NPCDef_Sploder` | `BP_Sploder` | `SK_Sploder(_Real)` | `MI_Sploder_Body_Own`, `MI_Sploder_Own` | — | `enemy_sploder_behavior` |
| Sword Enemy | `NPCDef_SwordEnemy` | `BP_SwordEnemy` | `CityofBrass_Enemies/.../Sword` | — | — | `enemy_sword_behavior` |
| Wizard | `NPCDef_Wizard` | `BP_Wizard` | `SK_Wizard(_Real)` | `MI_Wizard_Own` | — | `enemy_mage_behavior` |

## CityofBrass enemy mesh folders

```
CityofBrass_Enemies/Meshes/Enemy/
  Archer/  Charger/  Corpse/  Corpse_Basic/  Corpse_Melee/
  Corpse_Spear/  Grenadier/  Hooker/  Large/  Large_Charger/
  Large_Sword/  Sploder/  Sword/  Wizard/
```

Restored anims (examples): `CityofBrass_Enemies/Animations_Restored/{Charger,Corpse,Corpse_Spear,Grenadier}/`.

## Behavior → combat style

| Behavior class | Style | Hurt path |
|----------------|-------|-----------|
| `roguelike_enemy_behavior` | Default melee | `EnemyMeleeHit` |
| `enemy_melee_behavior` | Close chase | `EnemyMeleeHit` |
| `enemy_sword_behavior` | Aggressive flurry | `EnemyMeleeHit` |
| `enemy_spear_behavior` | Mid poke + keep range | `EnemyMeleeHit` |
| `enemy_charger_behavior` | Sprint slam | `EnemyMeleeHit` |
| `enemy_large_charger_behavior` | Wind-up heavy slam | `EnemyMeleeHit` |
| `enemy_hooker_behavior` | Claw | `EnemyMeleeHit` |
| `enemy_boss_behavior` | Heavy slam | `EnemyMeleeHit` |
| `enemy_sploder_behavior` | Arm → AOE + self-kill | `EnemyAOEHit` + `EnemyApplyHitDamage` |
| `enemy_archer_behavior` | Prefer range + SG arrow | `EnemyFireSGProjectile` |
| `enemy_mage_behavior` | Prefer range + fireball VFX | `EnemyFireSGProjectile` + mage VFX maps |
| `enemy_grenadier_behavior` | Prefer range + throw | `EnemyFireSGProjectile` |

## Inspecting NPCDef field values

Per-definition health / anim slot assignments live inside `.uasset` binary. With the Roguelike project open and listener online:

```
get_asset_info({"asset_path": "/Roguelike/AI/NPCDef_Archer"})
get_dependencies({"asset_path": "/Roguelike/AI/NPCDef_Archer"})
get_skeletal_mesh_info({"asset_path": "…"})
```

Document structural wiring here; dump live Details when recreating a specific enemy.
