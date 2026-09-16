# Movement System Technical Reference

This document is the technical reference for JetpackFPS's traversal kit: the movement state machine, the jetpack, wall running, sliding, and the animation gait pipeline. It is based on a graph-level audit of the current Blueprint implementation and the `RootMovement` C++ plugin, and it intentionally documents both the *intended* design and the *verified current behavior* — including inconsistencies — so they're tracked rather than silently relied on.

Primary assets:

- `/Game/ThirdPerson/Blueprints/BP_ThirdPersonCharacter` — movement coordinator
- `/Game/FPS/CoreMechanics/JetPack/JetPackComponent` — jetpack fuel/thrust/audio
- `/Game/FPS/CoreMechanics/EPlayerMovementState` — the movement state enum
- `/Game/FPS/CoreMechanics/ThrustCurve` — jetpack thrust intensity curve
- `Plugins/RootMovement_5.5/Source/RootMovement/*` — root-motion application (see [its README](../JetpackFPS/Plugins/RootMovement_5.5/README.md))

> **Note on confidence:** Where a Blueprint node's execution pin was observed unwired in the graph export, this doc calls it out explicitly ("not proven to execute" / "dead or stale"). Treat those branches as unreliable until re-verified in the Blueprint editor.

---

## 1. Movement State Machine

### 1.1 State enum

`EPlayerMovementState` values, in enum order:

| Value | Name |
| :-: | :-- |
| 0 | Walking |
| 1 | Sprinting |
| 2 | Crouching |
| 3 | JetPacking |
| 4 | WallRunning |
| 5 | Sliding |
| 6 | Proning |
| 7 | AirBorne |

### 1.2 `ResolveMovementState`

The general-purpose state resolver, evaluated in priority order:

```
if WallRunning? AND CharacterMovement.IsFalling:
    → WallRunning
else if CharacterMovement.IsFalling:
    → AirBorne
else if CanSprint():
    → Sprinting
else if CanStand():
    → Walking
else:
    → Crouching
```

`CanSprint()`:

```
(SprintKeyDown? OR ForwardAxis >= 0.8)
AND CanStand()
AND NOT CharacterMovement.IsFalling
```

**Important consequences:**

- WallRunning only outranks AirBorne while `WallRunning?` is true *and* the character is still falling.
- AirBorne outranks Sprinting/Walking/Crouching.
- **JetPacking, Sliding, and Proning are never selected by `ResolveMovementState`.** They're entered directly via `SetMovementState` from input events, and can be silently overwritten if `ResolveMovementState` runs while their supporting flags/conditions aren't actively maintained (e.g. landing while jetpacking).

### 1.3 `SetMovementState`

The controlled state-change entry point:

1. If `NewMovementState == MovementState`, exit immediately (no-op).
2. Store the current state in `PrevMovementState`.
3. Assign `MovementState = NewMovementState`.
4. Call `OnMovementStateChange(PrevMovementState)`.
5. Run the new-state branch of the `Switch Movement State` macro.

| New state | Verified behavior |
| :-- | :-- |
| Walking | Calls `EndCrouch` |
| Sprinting | Calls `EndCrouch` |
| Crouching | Calls `EndProne` if the previous state was Proning; otherwise `BeginCrouch` |
| Sliding | Checks `JetPackComponent.CheckSlideFuel`; calls `BeginSlide` if valid, otherwise falls back to `ResolveMovementState` |
| Proning | Calls `BeginProne` |
| JetPacking / WallRunning / AirBorne | No dedicated branch beyond the common `OnMovementStateChange` handling |

There is also buffered-slide handling inline in `SetMovementState` (consuming a buffered slide request on state resolution), but several of its branches and debug prints have unwired execution pins — don't treat them as reliably active without re-checking the graph.

### 1.4 `OnMovementStateChange`

Applies the physical movement settings for the transition, in three steps: set `MaxWalkSpeed` for the new state → apply previous-state exit effects → apply new-state enter effects.

**Max walk speed by state:**

| State | Speed |
| :-- | :-: |
| Walking | `WalkSpeed` = 600 |
| Sprinting | `RunningSpeed` = 800 |
| Crouching | `CrouchSpeed` = 300 |
| JetPacking | `RunningSpeed` = 800 |
| WallRunning | `WallRunningSpeed` = 1000 |
| Sliding | 0 (velocity is driven manually + by root motion, not acceleration) |
| Proning | `ProningSpeed` = 150 |
| AirBorne | `RunningSpeed` = 800 |

**Exit effects (leaving a state):**

- **JetPacking:** `AirControl = 0.35`, `GravityScale = 1.75`
- **Sliding:** `GroundFriction = 8`, `BrakingDecelerationWalking = 2000`, calls `EndSlide`
- **AirBorne:** `AirControl = 0.35`, `GravityScale = 1.75` (two additional `BrakingDecelerationFalling` setters at 2500/1500 have unwired execution inputs — not proven to execute)

**Enter effects (entering a state):**

- **JetPacking:** `AirControl = 0.95`, `GravityScale = 0.85`
- **Sliding:** `Velocity = ActorForwardVector * RunningSpeed`, `GroundFriction = 0`, `BrakingDecelerationWalking = 500`
- **AirBorne:** `AirControl = 0.90`, `GravityScale = 1.75`, `MaxAcceleration = 1500`
- **WallRunning** settings (`AirControl = 1.0`, `GravityScale = 0.0`, plane constraint) are set directly in `BeginWallRun`/`EndWallRun`, not centralized here.

### 1.5 Capsule resizing & jump resets

Capsule half-height changes are *not* driven from `OnMovementStateChange` — they're driven by dedicated timelines in `BeginCrouch`/`EndCrouch`/`BeginProne`/`EndProne`, which also reposition the first-person arm mesh relative to `StandingCameraZoffset`/`StandingCapsuleHalfHeight`.

`ResetJumpCount` (sets `JetPackComponent.JumpCount = 0`) is called from `OnLanded` and `BeginWallRun` — **not** from every state transition.

### 1.6 Cross-system conflict rules

**ADS vs. sprint:** `IA_Aim` cancels an active sprint — if aiming starts while `MovementState == Sprinting`, it sets `bSprintCancelledByADS = true`, forces `MovementState = Walking`, then calls `SetAiming(true)`. `AutoTacticalSprint` refuses to re-engage sprint while `ViewmodelController.IsAiming` or `bSprintCancelledByADS` is true; the flag clears when aiming ends. A `MovementStateBeforeADS` variable exists but has no active usage — the previous sprint state is **not** restored automatically; sprint must be reacquired via normal conditions.

**Jetpack vs. normal resolution:** `IA_Jump`'s second jump activates the jetpack directly (`Activate JetPack` → `IsJetPacking = true` → `MovementState = JetPacking`), and release deactivates and calls `ResolveMovementState`. Since `ResolveMovementState` never checks `IsJetPacking`/`IsJetPackHeld`, this is a one-way manual insertion into the state machine, not a resolved priority.

**Wall-run entry:** requires `WallRunning? == false`, `CanSurfaceBeWallRan(Hit.ImpactNormal)`, `CharacterMovement.IsFalling`, and `AreRequiredKeysDown()` (`ForwardAxis > 0.01`, plus `RightAxis > 0.1` toward a left wall or `< -0.1` toward a right wall).

**Wall-run exit:** ends when required input releases, the maintenance trace misses, or the detected wall side no longer matches `WallRunSide`. Jumping while wall-running calls `EndWallRun(JumpedOffWall)`. The `Reason` parameter (`FallOff` vs. `JumpedOffWall`) exists but no branch currently uses it — every exit path runs identical cleanup.

**Slide:** `SetMovementState(Sliding)` gates on `JetPackComponent.CheckSlideFuel`; insufficient fuel falls back to `ResolveMovementState` instead of sliding. Crouch while sprinting requests a slide directly; crouch while airborne sets a **0.35s** slide buffer (`SlideBuffWindow`) consumed on landing if not expired.

**Power slide:** `IA_PowerSlide` checks `JetPackComponent.IsSliding` — if already sliding it only prints `"Alreadyslding"` (typo preserved from the source graph); otherwise it calls `TriggerPowerSldie` (typo preserved — this is the actual function name in the Blueprint), and completion calls `EndPowerSlide`.

**No explicit rule exists** for: jetpack cancelling a ledge slide, ledge slide cancelling jetpack, wall-run cancelling jetpack, slide transitioning directly into wall-run, or ADS storing/restoring the exact pre-aim movement state.

---

## 2. Jetpack System

### 2.1 Ownership

`JetPackComponent` is an `ActorComponent` on the player character. It owns activation/deactivation, fuel consumption/recharge, thrust-over-time, jetpack/power-slide timers, jetpack audio, root-motion propulsion, and power-slide activation. It caches its owner (`Owning Player`, cast to `BP_ThirdPersonCharacter`) in `Begin Play`.

**It does not own `MovementState`.** The character Blueprint still sets `MovementState = JetPacking` directly and also writes `JetPackComponent.IsJetPacking` itself — see [§6 Known Issues](#6-known-verified-issues).

### 2.2 Key variables

**Fuel**

| Variable | Default | Purpose |
| :-- | :-: | :-- |
| `CurrentJetpackFuel` | 1.0 | Current fuel |
| `MaxJetPackFuel` | 1.0 | Fuel cap |
| `FuelRechargeRate` | 0.33 | Added per recharge tick (every 0.2s) |
| `FuelDrainPerSecond` | 6.0 | Base drain rate/sec |
| `PowerSlideThrust` | 0.3 | Fuel cost of a power slide |
| `FuelBurnMultiplier` | 75.0 | Used by a secondary curve-based drain branch (see below — not proven wired to the live drain) |

**Jetpack state**

| Variable | Default | Purpose |
| :-- | :-: | :-- |
| `IsJetPackHeld` | false | Activation input is being held |
| `IsJetPacking` | false | Propulsion is considered active |
| `JetpackHoldDuration` | 0.0 | How long the current hold has lasted |
| `MaxHoldDuration` | 1.5 | Normalizes hold duration |
| `HoldAlpha` | 0.0 | `JetpackHoldDuration / MaxHoldDuration`, clamped 0–1 |
| `ThrustIntensity` | 0.0 | `ThrustCurve.GetFloatValue(HoldAlpha)` |

**Thrust**

| Variable | Default | Purpose |
| :-- | :-: | :-- |
| `BaseJetPackStrength` | 1500.0 | Root-motion force scalar |
| `JetpackHorizontalForce` | 500.0 | Legacy/unused force branch (see §2.6) |
| `JetpackVerticalForce` | 2.0 | Legacy/unused force branch |

**Timers** (all ~50 Hz unless noted)

| Timer | Function | Interval |
| :-- | :-- | :-: |
| `TM_DecreaseFuel` | `DecreaseFuel` | 0.020s |
| `TM_Thrust` | `JetPackThrust` | 0.020s |
| `TM_IncreaseFuel` | `IncreaseFuel` | 0.200s |
| `TM_RechargeDelay` | `FuelRecharge` | one-shot, 0.7s after deactivation |

### 2.3 Component Tick — the hold/curve pipeline

While `IsJetPackHeld && CurrentJetpackFuel > 0`, component Tick (registered `During Physics`) runs every frame:

```
JetpackHoldDuration = Clamp(JetpackHoldDuration + DeltaSeconds, 0, MaxHoldDuration)
HoldAlpha           = Clamp(JetpackHoldDuration / MaxHoldDuration, 0, 1)
ThrustIntensity      = ThrustCurve.GetFloatValue(HoldAlpha)
```

This is a **separate timing system** from the 0.02s thrust/drain timers below — Tick updates the curve-driven intensity, timers apply it. See §6 for the synchronization risk this creates.

### 2.4 Activation / deactivation

**`Activate JetPack`** (custom event):

```
IsJetPackHeld = true
JetpackHoldDuration = 0
if CurrentJetpackFuel > 0:
    spawn looping jetpack audio
    clear recharge timers
    start TM_DecreaseFuel, TM_Thrust
    IsJetPacking = true
else:
    print "no fuel"   # does not reliably reset IsJetPackHeld — see §6
```

**`Deactivate JetPack`** (custom event) — *intended* flow: fade audio, clear all active timers, `IsJetPackHeld = false`, `IsJetPacking = false`, start `TM_RechargeDelay` (0.7s) → `FuelRecharge`. **As currently graphed, this function instead sets both flags to `true`** — see [§6](#6-known-verified-issues).

### 2.5 Fuel drain and recharge

Non-sliding drain, run by `TM_DecreaseFuel` every 0.02s:

```
BaseDrainPerTick = FuelDrainPerSecond * 0.020        # = 0.12 at defaults
ActualDrain      = BaseDrainPerTick * ThrustIntensity
CurrentJetpackFuel = Clamp(CurrentJetpackFuel - ActualDrain, 0, MaxJetPackFuel)
```

At full thrust intensity this is ≈ 6 fuel units/sec (i.e. a full 1.0 fuel tank drains in ~1 second at max thrust). A second branch multiplying `Curve(HoldAlpha) * FuelBurnMultiplier` (75.0) exists in the graph but is not proven to reach the live fuel setter — treat it as dead/parallel logic until re-verified.

Sliding drain (when `IsSliding`):

```
CurrentJetpackFuel = Clamp(CurrentJetpackFuel - PowerSlideThrust, 0, MaxJetPackFuel)   # PowerSlideThrust = 0.3
```

If fuel hits zero, the graph shows calls to `Deactivate JetPack` / `StopJumping`, but these have unwired execution inputs in `DecreaseFuel` — not proven to execute.

Recharge, gated by `FuelRecharge` checking `IsJetPacking == false`:

```
IncreaseFuel every 0.200s:
    CurrentJetpackFuel = Clamp(CurrentJetpackFuel + FuelRechargeRate, 0, MaxJetPackFuel)  # += 0.33
```

Recharge stops naturally once fuel clamps at `MaxJetPackFuel`, and does not start at all while `IsJetPacking` is true.

### 2.6 Thrust application — `JetPackThrust`

Runs every 0.02s via `TM_Thrust`. Direction:

```
Horizontal = ActorRightVector * RightAxis + ActorForwardVector * ForwardAxis
Vertical   = ActorUpVector * 0.75
WorldDirection = Normalize(Horizontal + Vertical)
Strength = BaseJetPackStrength * ThrustIntensity     # 1500 * ThrustIntensity
```

This is applied via `URootMovementLibrary::ApplyRootMotionConstantForce` with:

```
Duration = 0.020, IsAdditive = false, EnableGravity = false,
VelocityOnFinishMode = Maintain Last Root Motion Velocity, ClampVelocityOnFinish = 0
```

**Important — two curve-sampling conventions coexist in this function:** the primary path uses the Tick-computed `ThrustIntensity` (sampled via `HoldAlpha`, 0–1), but additional `GetFloatValue` nodes elsewhere in this graph sample the curve directly using raw `JetpackHoldDuration` (which ranges 0–1.5, not 0–1). If `ThrustCurve` is authored over a normalized 0–1 domain, that second path can sample outside the curve's intended range.

**Legacy/inactive branches** in this function — `LaunchCharacter`, `AddForce`, and debug line draws — have unwired execution inputs in the current graph and are not proven to run. `JetpackHorizontalForce`/`JetpackVerticalForce` are not proven to affect the live root-motion pins. Treat these as dead code until manually re-verified.

### 2.7 Root motion lifecycle (C++)

`URootMovementLibrary::ApplyRootMotionConstantForce` (`Plugins/RootMovement_5.5/Source/RootMovement/Public/RootMovementLibrary.h`) is a plain static Blueprint function: it builds an `FRootMotionSource_ConstantForce`, applies it via `CharacterMovement->ApplyRootMotionSource(...)`, and **discards the returned ID**. Each 0.02s jetpack-thrust tick therefore applies a brand-new override root-motion source with no stored handle, no explicit removal call, and no completion callback — its "lifecycle" is entirely implicit, relying on the 0.02s `Duration` matching the 0.02s timer interval.

Contrast with the **ledge-slide** path, which uses the async, callback-capable `UAsyncRootMovement : public UCancellableAsyncAction` (`AsyncRootMovement.h/.cpp`). `Activate()` stores the returned `RootMotionSourceID`, starts a one-shot completion timer, and `Cancel()` calls `RemoveRootMotionSourceByID`. **Note:** `Cancel()` dereferences `CharacterMovement` without a null check, even though `Activate()`'s failure path (`OnFail`) can be reached with an invalid `CharacterMovement` pointer — a latent crash risk if that path is exercised. The Blueprint's `OnComplete`/`OnFail` delegate pins for this async node were not resolvable as active event strands in the exported graph — the async node's callbacks are not confirmed to be consumed by any Blueprint logic; `TriggerPowerSldie` binds montage callbacks instead.

### 2.8 Power slide (jetpack-driven)

`CheckSlideFuel` gates on a `JumpCount` condition AND `CurrentJetpackFuel > 0.3`.

`TriggerPowerSldie` (event, name as authored — not a typo you need to fix elsewhere in code that references it): validates fuel → spawns slide audio → `isSliding = true` → one-shot `DecreaseFuel` → clears recharge timers → starts an async root-motion burst (`Strength = 800`, `Duration = 0.3s`, `Additive = true`, `ClampVelocityOnFinish = 600`, `EnableGravity = false`) → plays the `AM_Slide` montage → calls `EndCrouch` on a montage notify → calls `ResolveMovementState`.

`EndPowerSlide` clears all jetpack/slide timers and starts a fresh 0.7s `RechargeDelay`.

There is also a separate, apparently unused `PowerSlide` function on `JetPackComponent` that calls `LaunchCharacter(ActorForwardVector * (1500, 1500, -250))` — `TriggerPowerSldie` does not call it, so treat it as legacy/dead code.

---

## 3. Wall Running

### 3.1 Entry

Triggered by `CapsuleComponent → ComponentHit` (event-driven, not a continuous trace). Requires: not already wall-running, `CanSurfaceBeWallRan(Hit.ImpactNormal)`, `CharacterMovement.IsFalling`, and `AreRequiredKeysDown()`.

**`CanSurfaceBeWallRan`:** rejects `SurfaceNormal.Z < -0.05`; otherwise flattens the normal onto the horizontal plane, normalizes it, computes the angle between the original and flattened normals (`Acos` of their dot product), and accepts it if that angle is **less than** `CharacterMovement.GetWalkableFloorAngle()` — i.e. it accepts vertical wall-like surfaces and rejects floor-like ones. There is no separate velocity/approach-angle gate and no velocity projection onto the wall plane at entry.

**`FindRunDirectionAndSide`:** `DotProduct2D(WallNormalXY, ActorRightVectorXY) > 0` → side = Right, else Left. Run direction = `Cross(WallNormal, (0,0,∓1))` (sign depends on side). No explicit re-normalization after the cross product is shown — relies on the engine-normalized hit normal.

### 3.2 Maintenance

`UpdateWallRun` runs on the wall-run Timeline's update event (not confirmed to be on the main character Tick):

1. Re-check `AreRequiredKeysDown()` → `EndWallRun(FallOff)` if false.
2. `LineTraceSingle` (Visibility channel, `TraceComplex = false`, `IgnoreSelf = true`) from `ActorLocation` to `ActorLocation + Cross(WallRunDirection, SideVector) * 200`. This is a **line** trace, not a shape trace.
3. On miss → `EndWallRun(FallOff)`.
4. On hit: recompute direction/side from the new impact normal; if the recomputed side no longer matches the stored `WallRunSide` → `EndWallRun(FallOff)`.
5. Otherwise: `VelocityXY = WallRunDirection * CharacterMovement.GetMaxSpeed()`, `Velocity.Z = 0` — velocity is overwritten wholesale each update, not blended.

### 3.3 Entry/exit side effects

**On entry (`BeginWallRun`):** `MovementState = WallRunning`, play start SFX, spawn+fade in looping footstep SFX (0.8s), `ResetJumpCount`, `AirControl = 1.0`, `GravityScale = 0.0`, `SetPlaneConstraintNormal((0,0,1))`, `WallRunning? = true`, start camera tilt, call `UpdateWallRun`.

**On exit (`EndWallRun`):** fade loop audio (0.3s), play end SFX, `ResolveMovementState`, restore `AirControl = 0.35`, `GravityScale = 1.0`, clear plane constraint (`(0,0,0)`), `WallRunning? = false`, end camera tilt/timeline. **`EWallRunEndReason` (`FallOff`/`JumpedOffWall`) is accepted as a parameter but not currently branched on** — both reasons run identical cleanup.

**Plane constraint caveat:** the `(0,0,1)` constraint normal locks movement to a horizontal plane (removes vertical motion) — it is **not** derived from the wall's own normal. The illusion of running "on" the wall is maintained entirely by the repeated maintenance trace + velocity overwrite, not by a true wall-aligned constraint.

### 3.4 Camera tilt

`BeginCameraTilt`/`EndCameraTilt` drive a Timeline whose `Camera Roll` output is multiplied by `Select(WallRunSide)` (`Left = -1`, `Right = 1`) and applied as roll only — pitch/yaw are preserved from controller rotation. The tilt is a function of wall **side** and a curve, not of the wall normal, run direction, or relative velocity.

### 3.5 Hardcoded constants

| Constant | Value |
| :-- | :-: |
| Wall maintenance trace distance | 200 |
| Forward input threshold | 0.01 |
| Side input threshold | 0.1 |
| Downward-normal rejection (`SurfaceNormal.Z`) | -0.05 |
| Plane constraint normal | (0, 0, 1) |
| Horizontal wall-run velocity | exactly `GetMaxSpeed()` |
| Vertical wall-run velocity | forced to 0 |

Not currently implemented: an approach-angle dot-product gate, velocity projection onto the wall plane, wall-normal-derived camera tilt, shape-trace-based wall distance correction, or smoothing between changing wall normals.

---

## 4. Slide System

Two independent entry paths converge on the same `Sliding` state:

- **Sprint-to-slide:** crouch pressed while sprinting → `SetMovementState(Sliding)` → `BeginSlide`.
- **Jetpack power slide:** `IA_PowerSlide` → `JetPackComponent.TriggerPowerSldie` (see §2.8) directly, independent of `SetMovementState`.

### 4.1 `BeginSlide`

Checks `IsLedgeBelow` (a `SphereTraceSingle`, radius 2, offset 50 units forward, at capsule-derived vertical positions + 10 units) to choose between two slide variants:

**Standard slide** (no ledge): `MovementState = Sliding` → show jetpack widget → `TriggerPowerSldie` → play `SlideTimeline`. On each Timeline update: read `CurrentFloorHitResult.ImpactNormal` → `CalculateFloorInfluence` → apply via `CharacterMovement.AddForce`. Speed is clamped to `RunningSpeed`; once `VSize(Velocity) < CrouchSpeed`, the slide ends via `ResolveMovementState`.

**`CalculateFloorInfluence(FloorNormal)`:**

```
if FloorNormal == UpVector: return 0
tangent = normalize(cross(cross(FloorNormal, UpVector), FloorNormal))
SlopeFactor = Clamp(1 - Dot(FloorNormal, UpVector), 0, 1)
force = tangent * SlopeFactor * 1,000,000
```

This is a slope-scaled force model, not simple friction — the `1,000,000` multiplier is the dominant tuning constant here and should be treated as load-bearing, not incidental.

**Ledge slide** (ledge detected): `MovementState = Sliding` → a short async root-motion burst (`Strength = 500`, `Duration = 0.05s`, `Additive = true`, `EnableGravity = true`, forward direction) → `TriggerPowerSldie` → same speed cap / low-speed exit as the standard slide.

### 4.2 Standard vs. ledge slide

| | Standard slide | Ledge slide |
| :-- | :-- | :-- |
| Propulsion | `CalculateFloorInfluence` force via `AddForce` | 50ms additive root-motion burst |
| Gravity | Normal | Explicitly enabled on the root-motion source |
| Movement state | `Sliding` | `Sliding` (same state — no separate enum value) |

---

## 5. Animation & Gait Integration

`BP_ThirdPersonCharacter` implements `BPI_FPS_Viewmodel_C` and exposes pure getters (`GetCameraAnimator`, `GetViewmodelController`, `GetFirstPersonCamera`, `GetPlayerMesh`, `GetWeaponMesh`) — but coupling to `ViewmodelController` is largely **direct**, not interface-mediated: the character reads `ViewmodelController.IsAiming`, and calls `SetAimingStatus`, `PlayIKMotion`, and writes `Gait` directly.

**Gait pipeline**, run every Tick via `UpdateKinemationLocomotion`:

1. `UpdateMoveIntensity` — `Clamp(Size((RightAxis, ForwardAxis)), 0, 1)`.
2. `MapStateToTargetGait`:

   | State | Target gait |
   | :-- | :-: |
   | Walking | 1.0 |
   | Sprinting | 3.0 |
   | Crouching | 2.0 |
   | WallRunning | 2.0 |
   | JetPacking | 0.0 |
   | Sliding | 0.0 |
   | Proning | 0.0 |
   | AirBorne | 0.0 |

   Move intensity below `0.05` forces target gait to `0.0` regardless of state.
3. `ApplyKinemationGait` — `CurrentGait = FInterpTo(CurrentGait, TargetGait, DeltaSeconds, 10)`, written into `ViewmodelController.Gait`.

**Note:** a second function, `UpdateFirstPersonMovement`, also writes a raw (non-interpolated, non-state-mapped) value into `ViewmodelController.Gait`. It's currently reported as execution-unwired in the character's `EventGraph`, so it appears inactive — but if it's ever wired up, it and `UpdateKinemationLocomotion` would become competing writers of the same property. That function also contains an `FInterpTo` node with `Interp Speed = 0.0`, which — if active — cannot interpolate toward its target; treat this as a verified graph inconsistency rather than working logic.

Wall-run and slide camera tilt are driven directly by `BP_ThirdPersonCharacter` (`BeginCameraTilt`/`EndCameraTilt`, `BeginCameraSlideTilt`/`EndCameraSlideTilt`), not by `CameraAnimator_C` — the character owns that logic even though it also owns a `CameraAnimator` component reference.

Per-frame, `Tick` also runs `AutoTacticalSprint` (walking/sprinting only), `ClampHorizontalVelocity` (wall-running only — divides horizontal velocity down to `CharacterMovement.GetMaxSpeed()` when falling and over-speed), and maps current horizontal speed (0–1000) to a crosshair spread (5–45), smoothed via `FInterpTo`.

---

## 6. Known Verified Issues

These are graph-verified inconsistencies in the current implementation, ranked roughly by severity:

1. **`Deactivate JetPack` sets the wrong values.** The current graph sets `IsJetPackHeld = true` and `IsJetPacking = true` on deactivation (should be `false`/`false`). Reproduced failure: fuel recharges to 1.0, but a subsequent activation doesn't start thrust — `ActualDrain` stays `0.0` and the thrust/fuel timers never populate.
2. **The no-fuel activation branch doesn't fully reset state.** `Activate JetPack`'s no-fuel path clears `IsJetPackHeld` but does not clearly clear `IsJetPacking`.
3. **Jetpack state has multiple writers.** `JetPackComponent` sets `IsJetPacking` internally; `BP_ThirdPersonCharacter`'s `IA_Jump` graph *also* sets `JetPackComponent.IsJetPacking` and `MovementState` directly. There is no single source of truth.
4. **`ResolveMovementState` is jetpack-blind**, so any call to it (landing, slide-speed falloff, wall-run exit) can overwrite an active `JetPacking` state without deactivating the component first — `On Landed` in particular does not explicitly deactivate the jetpack before forcing `Walking`.
5. **Two curve-input conventions** for `ThrustCurve` (`HoldAlpha` vs. raw `JetpackHoldDuration`) coexist in `JetPackThrust` — see §2.6.
6. **No `RootMotionSourceID` lifecycle** for the repeating jetpack thrust source — see §2.7.
7. **Wall-run's plane constraint is not wall-aligned** — it's a fixed horizontal plane, see §3.3.
8. **`UAsyncRootMovement::Cancel()`** dereferences `CharacterMovement` without a null check, despite `Activate()`'s failure path being reachable with a null pointer.
9. **Several Blueprint branches are unwired / dead:** `LaunchCharacter` and `AddForce` in `JetPackThrust`; two `BrakingDecelerationFalling` setters in `OnMovementStateChange`; parts of the buffered-slide logic in `SetMovementState`; the legacy `PowerSlide` function on `JetPackComponent`; multiple `DrawDebugLine` calls; a duplicate `IsSliding = false` path; `UpdateFirstPersonMovement`'s zero-speed `FInterpTo`.
10. **Runtime debug `PrintString` nodes remain wired** in several slide/state-transition paths (`"normal slide"`, `"ledge sldie"`, `"Alreadyslding"`, `"no fuel"`, `"launch"`, `"buffered slide"`) — remove or gate these before a production/tutorial build.
11. **Not multiplayer-ready.** The character replicates (`bReplicates = true`, `bReplicateMovement = true`), but `JetPackComponent` does not (`bReplicates = false`), and none of `MovementState`, `IsJetPacking`, `IsJetPackHeld`, `IsSliding`, `CurrentJetpackFuel`, `WallRunning?`, `WallRunSide`, or `WallRunDirection` are replicated or RepNotify. Input directly drives local state; fuel timers and thrust application are local-only.

---

## 7. Tuning Constants Reference

| Constant | Value | Where |
| :-- | :-: | :-- |
| Jetpack thrust/drain timer interval | 0.020s | `JetPackComponent` |
| Fuel recharge delay | 0.700s | `JetPackComponent` |
| Fuel recharge tick interval | 0.200s | `JetPackComponent` |
| `FuelDrainPerSecond` | 6.0 | `JetPackComponent` |
| `FuelRechargeRate` | 0.33 | `JetPackComponent` |
| Power-slide fuel cost | 0.300 | `JetPackComponent` |
| Max jetpack hold duration | 1.500s | `JetPackComponent` |
| Base jetpack root-motion strength | 1500.0 | `JetPackComponent` |
| Jetpack vertical direction bias | 0.75 | `JetPackComponent` |
| Wall maintenance trace distance | 200 | `BP_ThirdPersonCharacter` |
| Wall forward input threshold | 0.01 | `BP_ThirdPersonCharacter` |
| Wall side input threshold | 0.1 | `BP_ThirdPersonCharacter` |
| Wall downward-normal rejection | -0.05 | `BP_ThirdPersonCharacter` |
| Ledge-slide root-motion strength / duration | 500 / 0.050s | `BP_ThirdPersonCharacter` |
| Power-slide async root-motion strength / duration | 800 / 0.300s | `JetPackComponent` |
| Standard slide braking deceleration | 500 | `BP_ThirdPersonCharacter` |
| Restored (post-slide) braking deceleration | 2000 | `BP_ThirdPersonCharacter` |
| Floor-influence force multiplier | 1,000,000 | `BP_ThirdPersonCharacter` |
| Slide input buffer window | 0.350s | `BP_ThirdPersonCharacter` |
| Gait interpolation speed | 10 | `BP_ThirdPersonCharacter` |
| Crosshair spread interpolation speed | 25 | `BP_ThirdPersonCharacter` |
| Wall-run / Jetpack / Airborne air control | 1.0 / 0.95 / 0.90 | `BP_ThirdPersonCharacter` |
| Wall-run / Jetpack / Airborne gravity scale | 0.0 / 0.85 / 1.75 | `BP_ThirdPersonCharacter` |

These values are currently spread across individual Blueprint node defaults rather than a single data asset — consolidating them into a `UDataAsset` (movement tuning table) would make balancing changes reviewable in source control instead of buried in graph diffs.
