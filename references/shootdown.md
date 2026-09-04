---
metadata:
  origin: store
---

# Shoot-down — how players destroy SG projectiles

Fortnite guns **cannot** damage Scene Graph spheres. The Roguelike path is: hold Fire + aim ray near the projectile → destroy + VFX.

Source: `enemy_projectile_shootdown.verse` + helpers in `enemy_combat_helpers.verse`.

## Device — place one in the level

```verse
enemy_projectile_shootdown<public> := class(creative_device):
    @editable KillVFX:?particle_system = false
    @editable MageFireballVFX:?particle_system = false
    @editable MageFireballImpactVFX:?particle_system = false

    OnBegin<override>()<suspends>:void =
        if (FX := KillVFX?):
            EnemyProjectileSetKillVFX(FX)
        if (FX := MageFireballVFX?):
            EnemyMageSetProjectileVFX(FX)
        if (FX := MageFireballImpactVFX?):
            EnemyMageSetImpactVFX(FX)
        PS := GetPlayspace()
        for (P : PS.GetPlayers()):
            BindPlayer(P)
        PS.PlayerAddedEvent().Subscribe(OnPlayerAdded)
        Print("enemy_projectile_shootdown READY — hold Fire + aim at arrows to destroy them")

    OnPlayerAdded<private>(P:player):void =
        BindPlayer(P)

    BindPlayer<private>(P:player):void =
        if (PI := GetPlayerInput[P]):
            PI.AddInputMapping(Character.RangedWeaponMapping)
            Events := PI.GetInputEvents(Character.WeaponPrimary)
            Events.BeginDetectEvent.Subscribe(OnFireBegin)
            Events.EndDetectEvent.Subscribe(OnFireEnd)

    OnFireBegin<private>(Result:tuple(player, logic)):void =
        EnemyProjectileSetFireHeld(Result(0), true)

    OnFireEnd<private>(Result:tuple(player, float)):void =
        EnemyProjectileSetFireHeld(Result(0), false)
```

## What it wires

| Details slot | Effect |
|--------------|--------|
| KillVFX | Session kill burst when projectile shot down (fallback `NS_ProjectileKill_asset`) |
| MageFireballVFX | Trail particles for mage casts (`EnemyMageGetProjectileVFX`) |
| MageFireballImpactVFX | Impact burst on mage hit/block |

## FireHeld map

```verse
var EnemyProjectileFireHeldMap<public>:weak_map(player, logic) = map{}
# Set true on WeaponPrimary BeginDetect; false on EndDetect
# EnemyFireSGProjectile only tests aim ray when FireHeld is true
```

**Trap:** if you test aim without FireHeld, looking at the archer destroys every inbound arrow on the same ray.

## Aim-ray math (in flight loop)

```verse
# Along = DotProduct(ToProj, ViewForward)
# Require Along > 80 and Along < MaxRange
# Closest point on ray within ShootDownRadius of projectile Pos → SHOT DOWN
```

Archer passes `?ShootDownRadius := ShootDownRadius` (default 100). Mage/grenadier use helper default 95 unless overridden.

## Kill VFX pattern

```verse
EnemyProjectilePlayKillVFX(Sim, Pos)<suspends>:
  SpawnParticleSystem(KillFX or NS_ProjectileKill_asset, Pos)
  Sleep(0.8)
  Cancel handle   # looping systems need Cancel
```

## Wiring checklist

1. Compile Verse with `enemy_projectile_shootdown` class.
2. Place device in level; label `EnemyProjectileShootdown`.
3. Assign KillVFX (+ mage VFX if using wizard).
4. Give players a ranged weapon / ensure RangedWeaponMapping works.
5. PIE: hold Fire while aiming at flying sphere → print `ARCHER SHOT DOWN` (or MAGE/GRENADIER).
