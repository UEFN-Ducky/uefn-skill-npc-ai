---
metadata:
  origin: store
---

**Tool order (HARD):** 1) Official UEFN MCP first (`ducky_get_status` → `epic_mcp_online` → nested `unreal__*`). 2) Ducky listener second. 3) `execute_python` LAST — never a placement path, even if Epic and listener failed. Map: `skill_read_subskill("uefn", "epic_mcp")`.

# Equipment on NPCs — swords, bows, custom weapons



Yes — enemies can hold / wear custom weapons. There are **two valid paths**. Pick one before authoring.



Cross-link (full MCP attach cookbook): `skill_read_subskill("animation", "npc_items")`.



## Attach (cosmetic) vs grant (usable inventory)



| Ask | Path |

|-----|------|

| "Sword **in his hand** / on his back" (looks right while chasing) | **This file** — mesh on skeleton socket, or baked into character mesh |

| "NPC can **shoot a Fortnite gun** / use inventory item" | Item Granter / NPC definition equipment / device wiring — **not** sockets. Combat damage for Roguelike enemies still goes through `EnemyMeleeHit` / `EnemyFireSGProjectile` (`projectiles`), not the gun. |



Roguelike melee/ranged **damage does not come from the held mesh**. The sword/bow is visual; hit logic is Verse.



---



## Path A — Baked into the character mesh (Roguelike default)



Many CityofBrass / Roguelike enemies already include the weapon in the skeletal mesh (or as a related mesh on the same skeleton):



| Enemy | Mesh evidence |

|-------|----------------|

| Corpse Sword | `CityofBrass_Enemies/Meshes/Enemy/Corpse/Corpse_Sword` (+ Skeleton / PhysicsAsset / Turban) |

| Sword / Large Sword | `Meshes/Enemy/Sword/`, `Meshes/Enemy/Large_Sword/`, `AI/EnemyMeshes/SK_LargeSword(_Real)` |

| Archer | `Meshes/Enemy/Archer/Archer` — bow often part of character / materials `MI_Archer_Bow`, `ArcherBow_Inst` |



**When to use:** marketplace enemy already has the weapon welded; you only need NPCDef + behavior.



**Recreate:** treat weapon as part of the character restore/retarget pipeline (`npc_characters`) — do **not** also socket-attach a second copy.



---



## Path B — Separate custom weapon mesh on a socket (your own swords / props)



Use when the weapon is a **standalone** static/skeletal mesh in the project (imported FBX, Meshy, Blender, etc.) and the character mesh has empty hands.



### Golden path (editor — one MCP op per call)



```

1. search_assets(search="sword", directory="/Game/…")     # FIND weapon in THIS project

2. get_asset_info(asset_path="…/SM_MySword")             # bounds / scale sanity

3. get_skeletal_mesh_info(asset_path="…/SK_Enemy")       # READ real bone names

4. list_skeleton_sockets(asset_path="…/SK_Enemy")        # reuse weapon_r / hand_r if present

5. add_skeleton_socket(                                  # CREATE slot if missing

     asset_path="…/SK_Enemy",

     bone_name="<real bone e.g. hand_r or Bip001-R-Hand>",

     socket_name="WeaponSocket_R",

     location=[0,0,0], rotation=[0,0,0])

6. spawn_actor(asset_path="…/SM_MySword")                # place weapon actor

7. attach_actor(child=sword, parent=npc_or_preview_bp,

     socket="WeaponSocket_R", rule="snap_to_target")

8. get_actor_bone_transform(... socket_or_bone="WeaponSocket_R")  # VERIFY

9. add_skeleton_socket(..., update_existing=true, …)     # iterate fit

10. save_current_level() / save_asset

```



### Bone name traps



- **Never guess** bones. Mannequin: `hand_r`, `hand_l`, `spine_03`, `head`.

- Biped / CityofBrass: often `Bip001-R-Hand`, `Bip001-Head`, …

- Read from `get_skeletal_mesh_info` / `list_skeleton_bones`.

- Prefer socket names by **slot** (`WeaponSocket_R`, `BackWeaponSocket`), not by item name.



### Custom weapon authoring tips



1. Import weapon into **active project** only (`/Game/AI/Weapons/…` or similar) — never outside the project folder.

2. Origin at the **grip** so socket offset stays near zero.

3. Compare weapon bounds vs character hand size before attach (10× sword is the classic fail).

4. Materials: project `MI_*` instances; assign on the weapon mesh.

5. Optional: duplicate Roguelike `Player_Sword_Mesh_Skeleton` / sword textures as reference scale only — still save new assets under `/Game/…` in **this** project.



---



## Runtime-spawned NPCs (`npc_spawner_device` / NPCDef)



Sockets live on the **Skeleton asset**, so every mesh sharing that skeleton has the socket — including runtime spawns.



But editor `attach_actor` only affects **actors placed in the level**. For spawner NPCs:



| Approach | When |

|----------|------|

| **Bake / merge** weapon into character mesh (or use Path A marketplace mesh) | Best for always-on sword/bow enemies |

| **Costume / mesh part** on the NPC definition if your pipeline supports it | Prefer over tick-follow |

| **Verse follow prop** | Last resort: `SpawnProp` + per-tick `MoveTo`/teleport to hand — one-frame lag; confirm APIs with `search_verse_digest({"query":"attach"})` / `get_verse_api` — **do not invent** attach APIs |



There is **no** reliable Fortnite Verse "AttachMeshToSocket on npc_behavior" in the Roguelike stack — combat uses Verse hits, visuals use content.



---



## Roguelike wiring checklist (sword enemy with custom blade)



1. Character mesh + physics + AnimPreset + `NPCDef_*` with `enemy_sword_behavior` (or melee) — see `roster` / `behaviors_melee`.

2. **If custom separate blade:** add `WeaponSocket_R` on that character's Skeleton → attach for **preview BP**; for runtime, bake blade into mesh or accept follow-prop fallback.

3. AttackAnim on VerseBehavior slots (swing clip) — damage still `EnemyMeleeHit` after anim.

4. Spawner → spawn manager — `spawning`.

5. PIE: sword visible on chase + `MELEE HIT` prints (mesh alone does not deal damage).



## Archer / ranged visual



- Bow: usually Path A (part of Archer mesh / materials).

- Fired "arrow" is **not** the held prop — Roguelike uses SG `EnemyFireSGProjectile` (`projectiles`). `ArrowProp` on archer behavior is a legacy NPCDef slot and unused for flight.



## Hard safety



- **Never** construct `unreal.SkeletalMeshSocket()` or touch `SubobjectDataSubsystem` via `execute_python` — native crash. Use `add_skeleton_socket` / `attach_actor` only.

- `remove_skeleton_socket` needs care — socket is shared by every mesh on that skeleton.

- All new meshes/materials stay inside the **active UEFN project** `/Game/…`.
