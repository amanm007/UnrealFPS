# RootMovement Plugin

Exposes Unreal's `FRootMotionSource_ConstantForce` root-motion source to Blueprints as two nodes — a fire-and-forget constant force, and an async, cancellable version with completion callbacks. This is the propulsion layer used by JetpackFPS's jetpack thrust and slide root-motion bursts (see [`docs/MOVEMENT_SYSTEM.md`](../../../docs/MOVEMENT_SYSTEM.md) for how the game code drives it).


## Why a plugin instead of raw velocity/impulse

Root motion sources (`FRootMotionSource_ConstantForce`) are applied through `UCharacterMovementComponent::ApplyRootMotionSource`, which integrates them alongside normal movement, network prediction, and other root-motion contributions (e.g. animation root motion) instead of stomping `Velocity` directly. That makes it a better fit for movement abilities like jetpack thrust and slide bursts than `AddForce`/`LaunchCharacter`, at the cost of needing to manage source IDs and durations explicitly — which is exactly what the async node below adds on top of the plain library function.

## API

### `URootMovementLibrary::ApplyRootMotionConstantForce`

**Blueprint node:** *Apply Root Motion Constant Force*

```cpp
static void ApplyRootMotionConstantForce(
    const UObject* WorldContext,
    UCharacterMovementComponent* CharacterMovement,
    FVector WorldDirection,
    float Strength,
    float Duration,
    bool bIsAdditive,
    UCurveFloat* StrengthOverTime,
    ERootMotionFinishVelocityMode VelocityOnFinishMode,
    FVector SetVelocityOnFinish,
    float ClampVelocityOnFinish,
    bool bEnableGravity
);
```

Builds an `FRootMotionSource_ConstantForce` (`Force = WorldDirection * Strength`, `Priority = 5`, `AccumulateMode` = Additive or Override per `bIsAdditive`) and applies it via `CharacterMovement->ApplyRootMotionSource(...)`. **The returned root-motion source ID is discarded** — this node is fire-and-forget: you get no completion signal and no way to explicitly cancel the source early. It's intended for short, self-expiring, repeatedly-reapplied forces (e.g. a thrust tick re-applied every frame/timer interval for as long as thrust should continue), not for a single long-duration force you might need to interrupt.

`bEnableGravity` sets `ERootMotionSourceSettingsFlags::IgnoreZAccumulate` when `false` — i.e. passing `bEnableGravity = false` makes the source override vertical accumulation (no gravity fighting the force); `true` lets normal gravity continue to accumulate alongside the force.

If `CharacterMovement` is null, the function logs a warning (`LogTemp`, Warning) and returns without applying anything — it does not crash.

### `UAsyncRootMovement`

**Blueprint node:** *Apply Root Motion Constant Force with Callbacks*

```cpp
static UAsyncRootMovement* AsyncRootMovement(
    const UObject* WorldContext,
    UCharacterMovementComponent* CharacterMovement,
    FVector WorldDirection,
    float Strength,
    float Duration,
    bool bIsAdditive,
    UCurveFloat* StrengthOverTime,
    ERootMotionFinishVelocityMode VelocityOnFinishMode,
    FVector SetVelocityOnFinish,
    float ClampVelocityOnFinish,
    bool bEnableGravity
);
```

Same parameters as the plain library call, but returns a `UCancellableAsyncAction` with two Blueprint-assignable delegates:

- `OnComplete` — fired when the force's `Duration` elapses naturally.
- `OnFail` — fired if the world or `CharacterMovement` is invalid when the action tries to activate.

Internally, `Activate()` applies the force and **stores** the returned `RootMotionSourceID`, then starts a one-shot timer for `Duration`; on expiry it broadcasts `OnComplete` and calls `Cancel()`. `Cancel()` calls `Super::Cancel()` and, if the completion timer is still valid, calls `CharacterMovement->RemoveRootMotionSourceByID(RootMotionSourceID)` to explicitly tear down the source — use this node (over the plain library call) whenever you need a guaranteed cleanup callback or the ability to cancel the force before it naturally expires (e.g. the player getting interrupted mid-slide).

> **Known issue:** `Cancel()` dereferences `CharacterMovement` without a null check. `Activate()`'s `OnFail` path can be reached with an invalid `CharacterMovement`, so a `Cancel()` call after that failure path is a potential null-pointer dereference. Guard this with a validity check before calling `RemoveRootMotionSourceByID` if you touch this code.

## Usage guidance

- Use `ApplyRootMotionConstantForce` for repeating, self-renewing forces where you re-issue the call on a fixed interval (as JetpackFPS does for jetpack thrust, reapplied every 0.02s) — you don't need a handle because each call's own `Duration` fully covers the gap until the next call.
- Use `AsyncRootMovement` for a single, longer burst you may need to react to or cancel mid-flight (as JetpackFPS does for its ledge-slide and power-slide bursts).
- Both nodes require a valid `UCharacterMovementComponent` — always resolve and null-check the character's movement component before calling either.
- `Priority = 5` is hardcoded in the plain library function; if you add other root-motion sources (e.g. montage root motion, other gameplay abilities) that need to win priority ties, account for this value.

## Building

This plugin is included with the `JetpackFPS` project and builds automatically as part of it — there's no separate build step. To use it in another project, copy the `RootMovement_5.5` folder into that project's `Plugins/` directory and enable it via the `.uproject` file or **Edit → Plugins**.
