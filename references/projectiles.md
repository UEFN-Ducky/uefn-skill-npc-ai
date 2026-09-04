---
metadata:
  origin: store
---

# Projectiles & damage — how NPCs hurt the player

**ONLY** place NPC combat may call `fort_character.Damage`. Behaviors must use `EnemyFireSGProjectile` / `EnemyMeleeHit` / `EnemyAOEHit`.

Source: `enemy_combat_helpers.verse`.

## How damage reaches the player

```mermaid
flowchart TD
  Behav[npc_behavior attack]
  Behav -->|melee after anim| Melee[EnemyMeleeHit]
  Behav -->|AOE explode| AOE[EnemyAOEHit]
  Behav -->|ranged| Proj[EnemyFireSGProjectile]
  Melee -->|still in range| Apply[EnemyApplyHitDamage]
  AOE -->|players in radius| Apply
  Proj -->|mid-flight HitRadius| Apply
  Apply --> Dmg["Target.Damage Amount"]
  Proj -->|Fire held + aim ray| ShotDown[Cancel + KillVFX]
  Proj -->|FindSweepHits wall| Blocked[Stop at wall]
```

## Centralized damage

```verse
EnemyApplyHitDamage<public>(Target:fort_character, Amount:float, Tag:string):void =
    if (Target.IsActive[]):
        Print("{Tag} dmg={Amount}")
        Target.Damage(Amount)

EnemyMeleeHit<public>(Attacker:fort_character, Target:fort_character, HitRange:float, Amount:float):logic =
    if (not Target.IsActive[]):
        return false
    Dist := Distance(Attacker.GetTransform().Translation, Target.GetTransform().Translation)
    if (Dist <= HitRange):
        EnemyApplyHitDamage(Target, Amount, "MELEE HIT dist={Dist}")
        return true
    Print("MELEE MISS dist={Dist} range={HitRange}")
    false

EnemyAOEHit<public>(Behavior:npc_behavior, Center:vector3, Radius:float, Amount:float):void =
    if (Ent := Behavior.GetEntity[], PS := Ent.GetPlayspaceForEntity[]):
        for (Player : PS.GetPlayers(), PChar := Player.GetFortCharacter[]):
            if:
                PChar.IsActive[]
                D := Distance(Center, PChar.GetTransform().Translation)
                D <= Radius
            then:
                EnemyApplyHitDamage(PChar, Amount, "AOE HIT dist={D}")
```

Melee only damages if **still** inside HitRange after the swing finishes.

## Shoot-down helpers (used by flight loop)

```verse
# True when Shooter's aim ray is within Radius of Pos.
# Call only while player is firing (FireHeld map) — looking alone must not delete arrows.
EnemyProjectileAimedAt<public>(Shooter:fort_character, Pos:vector3, Radius:float, MaxRange:float)<transacts>:logic =
    if (not Shooter.IsActive[]):
        return false
    ViewLoc := Shooter.GetViewLocation()
    Forward := Shooter.GetViewRotation().GetLocalForward()
    ToProj := Pos - ViewLoc
    Along := DotProduct(ToProj, Forward)
    if (Along > 80.0):
        if (Along < MaxRange):
            Closest := ViewLoc + Forward * Along
            if (Distance(Closest, Pos) <= Radius):
                return true
    false

var EnemyProjectileFireHeldMap<public>:weak_map(player, logic) = map{}

EnemyProjectileFireHeld<public>(Shooter:fort_character)<transacts>:logic =
    if (Agent := Shooter.GetAgent[], P := player[Agent]):
        if (EnemyProjectileFireHeldMap[P] = true):
            return true
    false

EnemyProjectileSetFireHeld<public>(Who:player, Held:logic):void =
    if (set EnemyProjectileFireHeldMap[Who] = Held) {}
```

Kill / mage VFX maps (session-scoped) are set by `enemy_projectile_shootdown` — see `shootdown`.

## `EnemyFireSGProjectile` — full recipe

Signature:

```verse
EnemyFireSGProjectile<public>(
    Behavior:npc_behavior,
    FromChar:fort_character,
    Target:fort_character,
    DamageAmount:float,
    Speed:float,
    HitRadius:float,
    SpawnHeight:float,
    AimHeight:float,
    StepDistance:float,
    ArrowScale:float,
    Tag:string,
    ?ShootDownRadius:float = 95.0,
    ?ProjectileVFX:?particle_system = false,
    ?ImpactVFX:?particle_system = false
)<suspends>:void
```

### Spawn + motion

1. Start = shooter pos + Z SpawnHeight; Aim = target pos + Z AimHeight.
2. Abort if ShotDist < 10.
3. Clearance nudge along Dir (min(100, ShotDist*0.35)) so sphere clears shooter.
4. `entity{}` + `sphere{ Collidable=true, Queryable=true, Visible=true }` + `keyframed_movement_component`.
5. `Sim.AddEntities` → `SetGlobalTransform` at StartNudge with `EnemyLookRotation`.
6. **Pre-sweep walls:** `FindSweepHits` along path; clamp EndPos to first hit past MinBlockDist=25.
7. One `keyframed_movement_delta` (translation = End-Start, Duration = FlyDist/Speed, linear easing) → `SetKeyframes` + `Play`.
8. Speed clamped ≥ 200; Scale ≥ 0.05; FlightTime ≥ 0.05.

### Per-frame loop (`Sleep(0.0)`)

- Refresh Pos from global transform.
- Trail VFX: every 4 frames cancel + respawn particle at Pos (static SpawnParticleSystem does not follow).
- If target dead → Stop + break.
- If Distance(Pos, AimNow) ≤ HitRadius → `EnemyApplyHitDamage(Target, DamageAmount, "{Tag} HIT (projectile)…")` → Stop.
- If ShootDownRadius > 0 and FireHeld and AimedAt → KillVFX, WasShotDown, Stop.
- If keyframes finished → final proximity check for hit; else miss/blocked print.

### Cleanup

- Cancel trail.
- Impact VFX if hit or blocked (not if shot down).
- Print BLOCKED / MISS.
- `ArrowEnt.RemoveFromParent()`.

### Exact body (copy into helpers module)

```verse
EnemyFireSGProjectile<public>(
    Behavior:npc_behavior,
    FromChar:fort_character,
    Target:fort_character,
    DamageAmount:float,
    Speed:float,
    HitRadius:float,
    SpawnHeight:float,
    AimHeight:float,
    StepDistance:float,
    ArrowScale:float,
    Tag:string,
    ?ShootDownRadius:float = 95.0,
    ?ProjectileVFX:?particle_system = false,
    ?ImpactVFX:?particle_system = false
)<suspends>:void =
    if (not Target.IsActive[]):
        return
    if (Ent := Behavior.GetEntity[], Sim := Ent.GetSimulationEntity[]):
        Start := FromChar.GetTransform().Translation + vector3{X := 0.0, Y := 0.0, Z := SpawnHeight}
        Aim := Target.GetTransform().Translation + vector3{X := 0.0, Y := 0.0, Z := AimHeight}
        ShotDist := Distance(Start, Aim)
        if (ShotDist < 10.0):
            return

        Dir := (Aim - Start) * (1.0 / ShotDist)
        var Clearance:float = 100.0
        if (Clearance > ShotDist * 0.35):
            set Clearance = ShotDist * 0.35
        StartNudge := Start + Dir * Clearance
        Rot := EnemyLookRotation(StartNudge, Aim)

        var ScaleVal:float = ArrowScale
        if (ScaleVal < 0.05):
            set ScaleVal = 0.05
        ScaleVec := vector3{X := ScaleVal, Y := ScaleVal, Z := ScaleVal}

        var FlySpeed:float = Speed
        if (FlySpeed < 200.0):
            set FlySpeed = 200.0

        ArrowEnt := entity{}
        Mesh := sphere{Entity := ArrowEnt}
        set Mesh.Collidable = true
        set Mesh.Queryable = true
        set Mesh.Visible = true
        MoveComp := keyframed_movement_component{Entity := ArrowEnt}
        ArrowEnt.AddComponents(array{Mesh, MoveComp})
        Sim.AddEntities(array{ArrowEnt})
        ArrowEnt.SetGlobalTransform(FromTransform(transform{Translation := StartNudge, Rotation := Rot, Scale := ScaleVec}))

        var FlyDist:float = ShotDist - Clearance
        if (FlyDist < 1.0):
            set FlyDist = 1.0
        var EndPos:vector3 = StartNudge + Dir * FlyDist
        MinBlockDist:float = 25.0
        ProbeXform := FromTransform(transform{Translation := StartNudge, Rotation := Rot, Scale := ScaleVec})
        ProbeDisp := FromVector3(EndPos - StartNudge)
        var HitWall:logic = false
        var HitAlong:float = FlyDist
        for (Hit : ArrowEnt.FindSweepHits(ProbeDisp, ProbeXform)):
            if (Hit.SourceHitDistance > MinBlockDist):
                if (Hit.SourceHitDistance < HitAlong):
                    set HitWall = true
                    set HitAlong = Hit.SourceHitDistance
        if (HitWall = true):
            set EndPos = StartNudge + Dir * HitAlong
            set FlyDist = HitAlong

        var FlightTime:float = FlyDist / FlySpeed
        if (FlightTime < 0.05):
            set FlightTime = 0.05

        TempDelta := transform{
            Translation := EndPos - StartNudge,
            Rotation := IdentityRotation(),
            Scale := vector3{X := 0.0, Y := 0.0, Z := 0.0}
        }
        KF := keyframed_movement_delta:
            Transform := FromTransform(TempDelta)
            Duration := FlightTime
            Easing := linear_easing_function{}
        MoveComp.SetKeyframes(array{KF}, oneshot_keyframed_movement_playback_mode{})
        Print("{Tag} FIRE speed={FlySpeed} dist={FlyDist} time={FlightTime}")
        MoveComp.Play()

        var DidHitPlayer:logic = false
        var WasBlocked:logic = HitWall
        var WasShotDown:logic = false
        var Pos:vector3 = StartNudge
        var TrailHandle:?cancelable = false
        var TrailCounter:int = 0
        if (FX := ProjectileVFX?):
            set TrailHandle = option{SpawnParticleSystem(FX, StartNudge, ?Rotation := Rot)}

        loop:
            Sleep(0.0)
            CurXform := FromTransform(ArrowEnt.GetGlobalTransform())
            set Pos = CurXform.Translation

            if (TrailHandle?):
                set TrailCounter = TrailCounter + 1
                if (TrailCounter >= 4):
                    set TrailCounter = 0
                    if (OldTrail := TrailHandle?):
                        OldTrail.Cancel()
                    if (FX := ProjectileVFX?):
                        set TrailHandle = option{SpawnParticleSystem(FX, Pos, ?Rotation := Rot)}

            if (not Target.IsActive[]):
                MoveComp.Stop()
                break

            AimNow := Target.GetTransform().Translation + vector3{X := 0.0, Y := 0.0, Z := AimHeight}
            HitDist := Distance(Pos, AimNow)
            if (HitDist <= HitRadius):
                EnemyApplyHitDamage(Target, DamageAmount, "{Tag} HIT (projectile) dist={HitDist}")
                set DidHitPlayer = true
                MoveComp.Stop()
                break

            if (ShootDownRadius > 0.0):
                if (EnemyProjectileFireHeld(Target) = true):
                    if (EnemyProjectileAimedAt(Target, Pos, ShootDownRadius, ShotDist + 400.0) = true):
                        Print("{Tag} SHOT DOWN")
                        spawn{EnemyProjectilePlayKillVFX(Sim, Pos)}
                        set WasShotDown = true
                        MoveComp.Stop()
                        break

            if (not MoveComp.IsPlaying[]):
                set Pos = EndPos
                AimFinal := Target.GetTransform().Translation + vector3{X := 0.0, Y := 0.0, Z := AimHeight}
                if (Distance(Pos, AimFinal) <= HitRadius):
                    EnemyApplyHitDamage(Target, DamageAmount, "{Tag} HIT (projectile)")
                    set DidHitPlayer = true
                break

        if (OldTrail := TrailHandle?):
            OldTrail.Cancel()

        if (WasShotDown = false):
            if (DidHitPlayer = true or WasBlocked = true):
                if (FX := ImpactVFX?):
                    spawn{EnemyProjectilePlayImpactVFX(FX, Pos)}

        if (DidHitPlayer = false):
            if (WasShotDown = false):
                if (WasBlocked = true):
                    Print("{Tag} BLOCKED by geometry")
                else:
                    AimNow := Target.GetTransform().Translation + vector3{X := 0.0, Y := 0.0, Z := AimHeight}
                    MissDist := Distance(Pos, AimNow)
                    Print("{Tag} MISS finalDist={MissDist}")
        ArrowEnt.RemoveFromParent()
```

## Walls requirement

`FindSweepHits` only sees Scene Graph mesh components with **Queryable=true**. Creative props / StaticMeshActors are invisible — use `SGSpawnBlockBox` / dungeon SG tiles / `sg_hub_collision_baker` (`spawning`).

## Expected PIE prints

- `MELEE HIT dist=…` / `MELEE MISS …`
- `AOE HIT dist=…`
- `ARCHER FIRE speed=…` → `ARCHER HIT (projectile)` / `MISS` / `BLOCKED` / `SHOT DOWN`
- `SPLODER SELF` after AOE
