---
metadata:
  origin: store
---

# Ranged archetypes — archer / mage / grenadier

All fire through `EnemyFireSGProjectile` (`projectiles`). No hitscan `Damage` in these behaviors.

## Shared keep-away pattern

```
Dist > MaxChase        → Sleep
Dist < PreferMin       → NavigateTo Away (PreferMax ring) Running
Dist <= AttackRange    → Focus + AttackAnim + EnemyFireSGProjectile + optional sidestep + Cooldown
else                   → NavigateTo PreferMin (Running or Walking)
```

## Defaults table

| Field | Archer | Mage | Grenadier |
|-------|--------|------|-----------|
| PreferMinRange | 900 | 700 | 800 |
| PreferMaxRange | 2200 | 1600 | 1800 |
| Attack range | ShootRange=2600 | CastRange=2000 | ThrowRange=2200 |
| Damage | 15 | 22 | 28 |
| Cooldown | 1.6 | 2.0 | 2.4 |
| Speed | ArrowSpeed=1600 | ProjectileSpeed=1200 | ProjectileSpeed=1000 |
| HitRadius | 70 | 80 | 100 |
| SpawnHeight | 90 | 100 | 90 |
| AimHeight | 70 | 70 | 40 |
| StepDistance | 48 | 40 | 40 |
| Scale | ArrowScale=0.28 | ProjectileScale=0.22 | ProjectileScale=0.28 |
| ShootDownRadius | 100 | (default 95 in helper) | (default 95) |
| MaxChaseRange | 8000 | 8000 | 8500 |
| NavReach | 120 | 120 | 120 |
| SidestepChance | 40 | 40 | 35 |
| Anim slot | AttackAnim | CastAnim | ThrowAnim |
| Tag | `"ARCHER"` | `"MAGE"` | `"GRENADIER"` |
| Extra VFX | — | MageProjectileVFX + ImpactVFX | — |

EnableSidestep=false, FocusYawOffsetDeg=0.0 on all.

---

## `enemy_archer_behavior` (exact fire call)

```verse
enemy_archer_behavior<public> := class(npc_behavior):
    @editable AttackAnim:?animation_sequence = false
    @editable ArrowProp:creative_prop_asset = DefaultCreativePropAsset  # unused for flight; keep for NPCDef slots
    @editable PreferMinRange:float = 900.0
    @editable PreferMaxRange:float = 2200.0
    @editable ShootRange:float = 2600.0
    @editable ShootDamage:float = 15.0
    @editable ShootCooldown:float = 1.6
    @editable ArrowSpeed:float = 1600.0
    @editable HitRadius:float = 70.0
    @editable ShootDownRadius:float = 100.0
    @editable SpawnHeight:float = 90.0
    @editable AimHeight:float = 70.0
    @editable MaxChaseRange:float = 8000.0
    @editable NavReach:float = 120.0
    @editable FocusYawOffsetDeg:float = 0.0
    @editable EnableSidestep:logic = false
    @editable SidestepChance:int = 40
    @editable StepDistance:float = 48.0
    @editable ArrowScale:float = 0.28

    # In ShootRange branch:
    #   PlayAndAwait(AttackAnim)
    #   EnemyFireSGProjectile(Self, Char, Target, ShootDamage, ArrowSpeed, HitRadius,
    #       SpawnHeight, AimHeight, StepDistance, ArrowScale, "ARCHER",
    #       ?ShootDownRadius := ShootDownRadius)
    #   optional sidestep NavigateTo
    #   Sleep(ShootCooldown)
```

Walls: SG Queryable colliders only (`sg_collision_helpers`, dungeon SG tiles).

---

## `enemy_mage_behavior`

```verse
enemy_mage_behavior<public> := class(npc_behavior):
    @editable CastAnim:?animation_sequence = false
    @editable PreferMinRange:float = 700.0
    @editable PreferMaxRange:float = 1600.0
    @editable CastRange:float = 2000.0
    @editable CastDamage:float = 22.0
    @editable CastCooldown:float = 2.0
    @editable ProjectileSpeed:float = 1200.0
    @editable HitRadius:float = 80.0
    @editable SpawnHeight:float = 100.0
    @editable AimHeight:float = 70.0
    @editable StepDistance:float = 40.0
    @editable ProjectileScale:float = 0.22
    @editable MaxChaseRange:float = 8000.0
    @editable NavReach:float = 120.0
    @editable FocusYawOffsetDeg:float = 0.0
    @editable EnableSidestep:logic = false
    @editable SidestepChance:int = 40

    # Fire call:
    EnemyFireSGProjectile(
        Self, Char, Target, CastDamage, ProjectileSpeed, HitRadius,
        SpawnHeight, AimHeight, StepDistance, ProjectileScale, "MAGE",
        ?ProjectileVFX := EnemyMageGetProjectileVFX(),
        ?ImpactVFX := EnemyMageGetImpactVFX()
    )
```

VFX assigned once via `enemy_projectile_shootdown` Details → `MageFireballVFX` / `MageFireballImpactVFX` (`shootdown`).

Approach outside CastRange uses **Walking**; back-off uses Running.

---

## `enemy_grenadier_behavior`

```verse
enemy_grenadier_behavior<public> := class(npc_behavior):
    @editable ThrowAnim:?animation_sequence = false
    @editable PreferMinRange:float = 800.0
    @editable PreferMaxRange:float = 1800.0
    @editable ThrowRange:float = 2200.0
    @editable ThrowDamage:float = 28.0
    @editable ThrowCooldown:float = 2.4
    @editable ProjectileSpeed:float = 1000.0
    @editable HitRadius:float = 100.0
    @editable SpawnHeight:float = 90.0
    @editable AimHeight:float = 40.0
    @editable StepDistance:float = 40.0
    @editable ProjectileScale:float = 0.28
    @editable MaxChaseRange:float = 8500.0
    @editable NavReach:float = 120.0
    @editable FocusYawOffsetDeg:float = 0.0
    @editable EnableSidestep:logic = false
    @editable SidestepChance:int = 35

    # Fire: EnemyFireSGProjectile(..., "GRENADIER") — no optional VFX args
```

Lower AimHeight (40) and larger HitRadius (100) = lobbed feel.

---

## Recreate checklist (ranged enemy)

1. NPCDef + `CharacterModifier_VerseBehavior` → this class.
2. Assign Attack/Cast/Throw anim.
3. Place `enemy_projectile_shootdown` in level (fire-held map + mage VFX).
4. Ensure dungeon/cover uses SG Queryable colliders.
5. PIE: expect prints `ARCHER FIRE …`, `ARCHER HIT (projectile)`, or `ARCHER MISS` / `BLOCKED` / `SHOT DOWN`.
