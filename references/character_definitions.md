---
metadata:
  origin: store
---

# NPCCharacterDefinition — composition

Pair with `skill_read_subskill("animation", "npc_characters")` for the **tools**.
This file is the composition; the animation pack is the create path.
**Never ask a human to author an NPCDef, AnimPreset, character BP, or fill Details.**

## Asset class

`/Script/VerseFortniteAI.NPCCharacterDefinition`

## Create it (tools, not Content Browser)

```
create_physics_asset_for_mesh({"skeletal_mesh_path": "…/SKM_*"})
create_anim_preset({"name": "AP_<Name>_Locomotion", "idle": "…/Idle", "walk": "…/Walk"})
create_character_blueprint({"name": "BP_<Name>", "skeletal_mesh_path": "…", "material_path": "…/MI_*", "scale": 2.0})
create_npc_character_definition({"name": "NPCDef_<Name>", "skeletal_mesh_path": "…",
  "character_blueprint_path": "…/BP_<Name>", "anim_preset_path": "…/AP_<Name>_Locomotion"})
workspace_compile_verse()
set_npc_definition_behavior({"asset_path": "…/NPCDef_<Name>", "behavior": "<npc_behavior class>"})
```

What `create_npc_character_definition` writes:

1. **Character type:** instanced `CharacterType_Custom`
2. **Skeletal mesh** with a **physics asset** (no physics → T-pose / slide)
3. **CosmeticSpawn:** `character_look = CHARACTER_BLUEPRINT`, `character_movement = ANIMATION_PRESET`, `support_anim_preset = true` — this is what makes a custom mesh spawn instead of a Fortnite skin
4. **Modifiers:** Health + CosmeticSpawn; VerseBehavior attached after compile
5. **`character_parts = []`** — populated parts re-enter the Fortnite outfit path
6. **`animation_bp = null`** — locomotion is the AnimPreset

Verify: `get_npc_definition_info` → `spawns_custom_mesh: true`.

## Reaction / attack clips — not Details

`set_verse_editable` cannot reach NPCDef VerseBehavior slots. Duplicate react clips into the **same folder as the behavior `.verse`** and reference them by identifier (`npc_ecosystem`). Do not tell the user to fill AttackAnim in Details.

Combat-enemy `@editable` numeric ranges still live on the class with defaults (see `behaviors_melee` / `behaviors_ranged`) — those do not need a human.

## Character Blueprint

`create_character_blueprint` — one mesh + material override + scale. Variants = more BPs, same mesh. Spawners use the NPCDef; CosmeticSpawn still needs the BP.

## Spawner wiring — not Details

1. Epic `PlaceDevice` Character Spawner (`npc_spawner_device`) per type. Label + folder in that call.
2. `set_npc_spawner_definition({"actor_path": "<label>", "definition_path": "…/NPCDef_*"})`
3. `wire_verse_device_ref` into the spawn manager / `cat_spawn_controller`.

## End-to-end (one new enemy)

1. Physics + retarget locomotion/attack onto one skeleton (`npc_characters`).
2. `create_anim_preset` idle/walk.
3. `create_character_blueprint` + `create_npc_character_definition`.
4. Duplicate attack clips into the Verse module folder; write or reuse the behavior class.
5. Compile → `set_npc_definition_behavior`.
6. Place spawner → `set_npc_spawner_definition` → wire manager → PIE.
