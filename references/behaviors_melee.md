---
metadata:
  origin: store
---

# Melee archetypes — exact defaults + loops

Source files: `enemy_melee_behavior.verse`, `enemy_charger_behavior.verse`, `enemy_sploder_behavior.verse`, `enemy_sploder_vfx_helpers.verse`, `enemy_hooker_behavior.verse`, `enemy_boss_behavior.verse`.

All damage goes through `EnemyMeleeHit` / `EnemyAOEHit` / `EnemyApplyHitDamage` (`projectiles`).

## Defaults table

| Class | Key fields (defaults) |
|-------|------------------------|
| `enemy_melee_behavior` | HitRange=220, HitDamage=20, HitCooldown=1.2, RetargetSecs=0.55, MaxChase=7500, NavReach=140, SpreadDist=260, SidestepChance=35 |
| `enemy_sword_behavior` | HitRange=250, HitDamage=28, HitCooldown=0.85, MaxChase=8500, NavReach=110, SpreadDist=220, SidestepChance=40; loop Sleep=0.08; retarget Sleep=0.4 |
| `enemy_spear_behavior` | PreferMin=350, PreferMax=700, PokeRange=520, PokeDamage=24, PokeCooldown=1.5, MaxChase=8000, SidestepChance=45 |
| `enemy_charger_behavior` | ChargeTriggerRange=1800 (unused in loop body), SlamRange=280, SlamDamage=40, SlamCooldown=2.2, MaxChase=9000, SpreadDist=240 |
| `enemy_large_charger_behavior` | SlamRange=360, SlamDamage=65, SlamCooldown=3.0, WindUpSecs=0.45, MaxChase=9500, SpreadDist=320; hit uses SlamRange*1.15 |
| `enemy_hooker_behavior` | HookRange=1200 (unused in loop), ClawRange=240, ClawDamage=25, ClawCooldown=1.1, MaxChase=8500, SpreadDist=200 |
| `enemy_boss_behavior` | SlamRange=320, SlamDamage=50, SlamCooldown=2.0, MaxChase=9500, SpreadDist=300 |
| `enemy_sploder_behavior` | ExplodeRange=320, ExplodeDamage=85, ArmRange=200, SelfKillDamage=9999, VFXLifetime=2.5, MaxChase=9000, NavReach=55, SpreadDist=180 |

EnableSidestep defaults **false** on all; FocusYawOffsetDeg = 0.0.

---

## `enemy_melee_behavior`

```verse
enemy_melee_behavior<public> := class(npc_behavior):
    @editable AttackAnim:?animation_sequence = false
    @editable HitRange:float = 220.0
    @editable HitDamage:float = 20.0
    @editable HitCooldown:float = 1.2
    @editable RetargetSecs:float = 0.55
    @editable MaxChaseRange:float = 7500.0
    @editable NavReach:float = 140.0
    @editable FocusYawOffsetDeg:float = 0.0
    @editable SpreadDist:float = 260.0
    @editable EnableSidestep:logic = false
    @editable SidestepChance:int = 35

    OnBegin<override>()<suspends>:void =
        if:
            Agent := GetAgent[]
            Char := Agent.GetFortCharacter[]
            Nav := Char.GetNavigatable[]
            Focus := Char.GetFocusInterface[]
            Anim := Char.GetPlayAnimationController[]
        then:
            loop:
                Sleep(0.1)
                if (not Char.IsActive[]):
                    break
                if (Target := EnemyFindNearestPlayer(Self, Char, MaxChaseRange)?):
                    SelfPos := Char.GetTransform().Translation
                    TargetPos := Target.GetTransform().Translation
                    Dist := Distance(SelfPos, TargetPos)
                    LookAt := EnemyLookPoint(SelfPos, TargetPos, FocusYawOffsetDeg)
                    if (Dist <= HitRange):
                        race:
                            Focus.MaintainFocus(LookAt)
                            block:
                                if (Clip := AttackAnim?):
                                    Anim.PlayAndAwait(Clip)
                                EnemyMeleeHit(Char, Target, HitRange, HitDamage)
                                Sleep(HitCooldown)
                    else if (Dist <= MaxChaseRange):
                        ShouldStrafe := EnemyShouldSidestep(EnableSidestep, SidestepChance)
                        if (ShouldStrafe = true):
                            Strafe := EnemyStrafePoint(SelfPos, TargetPos, 160.0, 420.0)
                            race:
                                Nav.NavigateTo(MakeNavigationTarget(Strafe), ?MovementType := movement_types.Running, ?ReachRadius := NavReach)
                                Focus.MaintainFocus(LookAt)
                                Sleep(RetargetSecs)
                        else:
                            Approach := EnemySpreadApproachPoint(SelfPos, TargetPos, SpreadDist, 180.0, 520.0)
                            race:
                                Nav.NavigateTo(MakeNavigationTarget(Approach), ?MovementType := movement_types.Running, ?ReachRadius := NavReach)
                                Focus.MaintainFocus(LookAt)
                                Sleep(RetargetSecs)
                    else:
                        Sleep(0.4)
                else:
                    Sleep(1.0)
```

## `enemy_sword_behavior` (same file)

Faster loop (Sleep 0.08), HitRange=250, HitDamage=28, HitCooldown=0.85, MaxChase=8500, NavReach=110, SpreadDist=220, SidestepChance=40, retarget Sleep=0.4, strafe 140–380, approach side 160–480. Same race/melee structure as melee.

## `enemy_spear_behavior` (same file)

Mid-range poke: back off if `Dist < PreferMinRange`; poke if `Dist <= PokeRange`; else approach PreferMin walking. After poke, optional sidestep then `Sleep(PokeCooldown)`.

```verse
# Distance branches (essentials):
# Dist > MaxChase → Sleep(0.5)
# Dist < PreferMin → NavigateTo Away at PreferMax (Running)
# Dist <= PokeRange → PlayAndAwait + EnemyMeleeHit(PokeRange, PokeDamage) + optional strafe + Sleep(PokeCooldown)
# else → NavigateTo PreferMin (Walking), Sleep(0.65)
```

---

## `enemy_charger_behavior`

```verse
enemy_charger_behavior<public> := class(npc_behavior):
    @editable SlamAnim:?animation_sequence = false
    @editable ChargeTriggerRange:float = 1800.0
    @editable SlamRange:float = 280.0
    @editable SlamDamage:float = 40.0
    @editable SlamCooldown:float = 2.2
    @editable MaxChaseRange:float = 9000.0
    @editable NavReach:float = 100.0
    @editable FocusYawOffsetDeg:float = 0.0
    @editable SpreadDist:float = 240.0
    @editable EnableSidestep:logic = false
    @editable SidestepChance:int = 30
    # OnBegin: Dist<=SlamRange → SlamAnim + EnemyMeleeHit; else strafe/approach Running
```

## `enemy_large_charger_behavior`

Wind-up before slam:

```verse
# Dist <= SlamRange:
#   Sleep(WindUpSecs)  # 0.45
#   PlayAndAwait(SlamAnim)
#   EnemyMeleeHit(Char, Target, SlamRange * 1.15, SlamDamage)
#   Sleep(SlamCooldown)  # 3.0
```

---

## `enemy_hooker_behavior`

ClawRange=240, ClawDamage=25, ClawCooldown=1.1. Same chase/slam structure as charger with `ClawAnim`.

## `enemy_boss_behavior`

SlamRange=320, SlamDamage=50, SlamCooldown=2.0, SpreadDist=300. Heavy slam lanes.

---

## Sploder — AOE + self-kill

### VFX helper (`enemy_sploder_vfx_helpers.verse`)

```verse
SploderPlayExplosionVFX<public>(Pos:vector3, MaybeFX:?particle_system, Lifetime:float)<suspends>:void =
    var Handle:?cancelable = false
    if (FX := MaybeFX?):
        set Handle = option{SpawnParticleSystem(FX, Pos)}
    else:
        set Handle = option{SpawnParticleSystem(NS_Sploder_Explosion_asset, Pos)}
    Sleep(Lifetime)
    if (H := Handle?):
        H.Cancel()
```

### Behavior essentials

```verse
enemy_sploder_behavior<public> := class(npc_behavior):
    @editable ExplodeAnim:?animation_sequence = false
    @editable ExplodeVFX:?particle_system = false
    @editable VFXLifetime:float = 2.5
    @editable BoomTrigger:?trigger_device = false
    @editable ExplodeRange:float = 320.0
    @editable ExplodeDamage:float = 85.0
    @editable ArmRange:float = 200.0
    @editable MaxChaseRange:float = 9000.0
    @editable NavReach:float = 55.0
    @editable SelfKillDamage:float = 9999.0
    @editable SpreadDist:float = 180.0
    # …
    # Dist <= ArmRange:
    #   PlayAndAwait ExplodeAnim (or Sleep 0.12)
    #   TriggerDeathExplosion(BlastPos)  # spawn VFX once
    #   BoomTrigger.Trigger() if set
    #   EnemyAOEHit(Self, BlastPos, ExplodeRange, ExplodeDamage)
    #   EnemyApplyHitDamage(Char, SelfKillDamage, "SPLODER SELF")
    # OnEnd: if VFX not yet played, spawn at LastKnownPos / char pos
```

**Hurt path:** every active player within `ExplodeRange` of blast center takes `ExplodeDamage`; sploder then self-damages to die.

Wire `SploderSpawner` separately on `enemy_spawn_manager` (see `spawning`).
