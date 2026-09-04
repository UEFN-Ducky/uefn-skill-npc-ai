---
metadata:
  origin: store
---

# Behavior base — helpers + default melee

Exact Roguelike code from `enemy_ai_helpers.verse` and `roguelike_enemy_behavior.verse`.

## Module layout

```
Verse/GameDevices/AISystems/
  enemy_ai_helpers.verse       # look / nearest / spread / sidestep
  roguelike_enemy_behavior.verse
  enemy_combat_helpers.verse   # see projectiles
```

## TEST TOGGLE — straight chase

When `EnemyStraightChaseTest = true` (current Roguelike default):

- `EnemySpreadApproachPoint` returns `TargetPos` (no weave)
- `EnemyShouldSidestep` always returns false

Set to `false` to restore pack spread + sidestep.

```verse
EnemyStraightChaseTest<public>:logic = true
```

## Look / nearest player

```verse
# Face From -> To (yaw/pitch) for flying props.
EnemyLookRotation<public>(From:vector3, To:vector3):rotation =
    Direction := To - From
    NLength := Distance(From, To)
    if (NLength < 0.001):
        IdentityRotation()
    else:
        N := Direction * (1.0 / NLength)
        YawRadians := ArcTan(N.Y, N.X) - (3.14159 / 2.0)
        Horiz := Distance(vector3{X := 0.0, Y := 0.0, Z := 0.0}, vector3{X := N.X, Y := N.Y, Z := 0.0})
        PitchRadians := ArcTan(N.Z, Horiz)
        YawDegrees := YawRadians * 180.0 / 3.14159
        PitchDegrees := PitchRadians * 180.0 / 3.14159
        MakeRotationFromYawPitchRollDegrees(YawDegrees, PitchDegrees, 0.0)

# Always aim at the target so NPCs face the player.
EnemyLookPoint<public>(SelfPos:vector3, TargetPos:vector3, YawOffsetDeg:float):vector3 =
    TargetPos

EnemyFindNearestPlayer<public>(Behavior:npc_behavior, FortChar:fort_character, DetectionRange:float)<transacts>:?fort_character =
    var Best:?fort_character = false
    var BestDist:float = DetectionRange
    if (Ent := Behavior.GetEntity[], PS := Ent.GetPlayspaceForEntity[]):
        SelfPos := FortChar.GetTransform().Translation
        for (Player : PS.GetPlayers(), PlayerChar := Player.GetFortCharacter[]):
            if:
                PlayerChar.IsActive[]
                not PlayerChar = FortChar
                D := Distance(SelfPos, PlayerChar.GetTransform().Translation)
                D < BestDist
            then:
                set Best = option{PlayerChar}
                set BestDist = D
    Best
```

## Spread / strafe / sidestep

```verse
EnemyXYLength<public>(V:vector3):float =
    Distance(vector3{X := 0.0, Y := 0.0, Z := 0.0}, vector3{X := V.X, Y := V.Y, Z := 0.0})

EnemyLateralDir<public>(SelfPos:vector3, TargetPos:vector3):vector3 =
    Dir := TargetPos - SelfPos
    Lat := vector3{X := -Dir.Y, Y := Dir.X, Z := 0.0}
    Len := EnemyXYLength(Lat)
    if (Len > 1.0):
        Lat * (1.0 / Len)
    else:
        vector3{X := 1.0, Y := 0.0, Z := 0.0}

EnemyRandomSide<public>():float =
    if (GetRandomInt(0, 1) = 0):
        -1.0
    else:
        1.0

EnemySpreadApproachPoint<public>(SelfPos:vector3, TargetPos:vector3, PreferDist:float, SideMin:float, SideMax:float):vector3 =
    if (EnemyStraightChaseTest = true):
        TargetPos
    else:
        Away := SelfPos - TargetPos
        AwayLen := EnemyXYLength(Away)
        var RingDir:vector3 = vector3{X := 1.0, Y := 0.0, Z := 0.0}
        if (AwayLen > 1.0):
            set RingDir = vector3{X := Away.X, Y := Away.Y, Z := 0.0} * (1.0 / AwayLen)
        var Lo:float = SideMin
        var Hi:float = SideMax
        if (Lo > Hi):
            set Lo = SideMax
            set Hi = SideMin
        if (Hi < 1.0):
            set Hi = 1.0
        Side := GetRandomFloat(Lo, Hi) * EnemyRandomSide()
        var RingDist:float = PreferDist + GetRandomFloat(-50.0, 90.0)
        if (RingDist < 40.0):
            set RingDist = 40.0
        Lat := EnemyLateralDir(SelfPos, TargetPos)
        TargetPos + RingDir * RingDist + Lat * Side

EnemyStrafePoint<public>(SelfPos:vector3, TargetPos:vector3, StrafeMin:float, StrafeMax:float):vector3 =
    Lat := EnemyLateralDir(SelfPos, TargetPos)
    Dist := GetRandomFloat(StrafeMin, StrafeMax)
    SelfPos + Lat * (EnemyRandomSide() * Dist)

EnemyChance<public>(ChancePercent:int):logic =
    if (GetRandomInt(1, 100) <= ChancePercent):
        true
    else:
        false

EnemyShouldSidestep<public>(EnableSidestep:logic, ChancePercent:int):logic =
    if (EnemyStraightChaseTest = true):
        false
    else if (EnableSidestep = true):
        EnemyChance(ChancePercent)
    else:
        false
```

## Default behavior — `roguelike_enemy_behavior`

Exact `@editable` defaults:

| Field | Default |
|-------|---------|
| AttackAnim | false |
| HitRange | 220.0 |
| HitDamage | 18.0 |
| HitCooldown | 1.3 |
| FocusYawOffsetDeg | 0.0 |
| SpreadDist | 260.0 |
| EnableSidestep | false |
| SidestepChance | 35 |

```verse
roguelike_enemy_behavior<public> := class(npc_behavior):
    @editable AttackAnim:?animation_sequence = false
    @editable HitRange:float = 220.0
    @editable HitDamage:float = 18.0
    @editable HitCooldown:float = 1.3
    @editable FocusYawOffsetDeg:float = 0.0
    @editable SpreadDist:float = 260.0
    @editable EnableSidestep:logic = false
    @editable SidestepChance:int = 35

    OnBegin<override>()<suspends>:void =
        if:
            Agent := GetAgent[]
            NpcChar := Agent.GetFortCharacter[]
            Nav := NpcChar.GetNavigatable[]
            Focus := NpcChar.GetFocusInterface[]
            Anim := NpcChar.GetPlayAnimationController[]
        then:
            loop:
                Sleep(0.1)
                if (not NpcChar.IsActive[]):
                    break
                if (Target := EnemyFindNearestPlayer(Self, NpcChar, 7500.0)?):
                    SelfPos := NpcChar.GetTransform().Translation
                    TargetPos := Target.GetTransform().Translation
                    Dist := Distance(SelfPos, TargetPos)
                    LookAt := EnemyLookPoint(SelfPos, TargetPos, FocusYawOffsetDeg)
                    if (Dist <= HitRange):
                        race:
                            Focus.MaintainFocus(LookAt)
                            block:
                                if (Clip := AttackAnim?):
                                    Anim.PlayAndAwait(Clip)
                                EnemyMeleeHit(NpcChar, Target, HitRange, HitDamage)
                                Sleep(HitCooldown)
                    else if (Dist <= 7500.0):
                        ShouldStrafe := EnemyShouldSidestep(EnableSidestep, SidestepChance)
                        if (ShouldStrafe = true):
                            Strafe := EnemyStrafePoint(SelfPos, TargetPos, 160.0, 420.0)
                            race:
                                Nav.NavigateTo(MakeNavigationTarget(Strafe), ?MovementType := movement_types.Running, ?ReachRadius := 150.0)
                                Focus.MaintainFocus(LookAt)
                                Sleep(0.55)
                        else:
                            Approach := EnemySpreadApproachPoint(SelfPos, TargetPos, SpreadDist, 180.0, 520.0)
                            race:
                                Nav.NavigateTo(MakeNavigationTarget(Approach), ?MovementType := movement_types.Running, ?ReachRadius := 150.0)
                                Focus.MaintainFocus(LookAt)
                                Sleep(0.55)
                    else:
                        Sleep(0.5)
                else:
                    Sleep(1.0)
```

## Skeleton every archetype shares

1. Guard `GetAgent` / `GetFortCharacter` / `GetNavigatable` / `GetFocusInterface` / `GetPlayAnimationController`
2. Loop: Sleep → dead break → nearest player
3. In range → `race { MaintainFocus ; PlayAndAwait + hit helper + cooldown }`
4. Out of range → NavigateTo(spread or strafe) raced with MaintainFocus + retarget Sleep
