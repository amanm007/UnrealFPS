# JetpackFPS

A first-person shooter movement sandbox built in **Unreal Engine 5.5**, centered on a fast, momentum-driven traversal kit: sprinting, sliding, wall-running, proning, and a fuel-limited jetpack with hold-based thrust ramping. Movement propulsion (jetpack thrust, ledge-slide bursts) is driven through a custom root-motion plugin rather than raw velocity/impulse hacks.

> The gameplay layer is implemented primarily in Blueprint (`BP_ThirdPersonCharacter`, `JetPackComponent`). The `Source/JetpackFPS` C++ module is currently just the standard module bootstrap — it does not yet contain the movement code. The one piece of hand-written C++ is the `RootMovement` plugin, which the jetpack and ledge-slide systems call into for root-motion force application.

## Features

- **8-state movement state machine** — Walking, Sprinting, Crouching, JetPacking, WallRunning, Sliding, Proning, AirBorne (`EPlayerMovementState`)
- **Jetpack** — hold-to-charge thrust curve, timer-driven fuel drain/recharge (~50 Hz), root-motion propulsion via the `RootMovement` plugin
- **Wall running** — capsule-hit-triggered entry, surface-angle validation against the walkable floor angle, wall-relative run direction via cross product, camera roll tilt
- **Sliding** — sprint-triggered slide with floor-slope-influenced force, plus a distinct ledge-slide using a short additive root-motion burst; slide input is buffered for ~0.35s so crouch-before-landing still triggers a slide
- **Power slide** — jetpack-fuel-gated slide burst with its own root-motion profile
- **Vaulting, proning, ADS/sprint conflict resolution, capsule-height timelines** for crouch/prone
- **Enhanced Input** for all movement actions (move, sprint, crouch, jump/jetpack, aim, power-slide)
- Gait-driven animation blending forwarded to a `ViewmodelController` component for first-person arm/weapon animation

## Requirements

- Unreal Engine **5.5** (project's `EngineAssociation`; some internal audit notes were captured against a 5.7.3 editor build — verify your target engine version before opening)
- Visual Studio 2022 (Windows) with the "Game development with C++" workload, or an equivalent toolchain for your platform
- Plugins enabled by the project: `ModelingToolsEditorMode` (editor-only), `RawInput`, `GameplayAbilities`, plus the bundled `RootMovement` plugin. `FPSAnimationPack` is present but disabled.

## Getting Started

1. Clone the repository.
2. Right-click `JetpackFPS/JetpackFPS.uproject` → **Generate Visual Studio project files** (or run `GenerateProjectFiles` for your platform).
3. Open `JetpackFPS.sln` and build the `Development Editor` configuration, or double-click the `.uproject` to let the engine build it for you.
4. Open the project in the editor and play `BP_ThirdPersonCharacter` in any level that contains a `PlayerStart`.

### Default controls (Enhanced Input)

| Action | Behavior |
| :-- | :-- |
| Move | `IA_Move` — standard WASD-style movement |
| Sprint (hold) | Enables sprint if `CanSprint()` passes (not falling, enough headroom to stand) |
| Crouch | Crouch while walking; slide while sprinting; buffers a slide if pressed while airborne |
| Jump | Jump/wall-jump; jump again in air (fuel permitting) to activate the jetpack; release to deactivate |
| Aim (ADS) | Cancels an active sprint (`bSprintCancelledByADS`) |
| Power Slide | Triggers a fuel-gated power slide via `JetPackComponent` |

## Project Structure

```
JetpackFPS/
├── JetpackFPS.uproject
├── Source/
│   └── JetpackFPS/            # C++ module bootstrap (movement logic lives in Blueprint, not here)
├── Plugins/
│   └── RootMovement_5.5/      # Custom root-motion plugin (see its own README)
├── Content/
│   ├── FPS/CoreMechanics/     # JetPackComponent, EPlayerMovementState, ThrustCurve
│   └── ThirdPerson/           # BP_ThirdPersonCharacter and related Blueprints
└── Config/
docs/
└── MOVEMENT_SYSTEM.md         # Full movement/jetpack/wall-run/slide technical reference
```

## Core Systems

The full technical reference — state machine priority rules, the jetpack fuel/thrust pipeline, wall-run vector math, the slide system, animation gait integration, tuning constants, and a list of known inconsistencies in the current implementation — lives in **[`docs/MOVEMENT_SYSTEM.md`](docs/MOVEMENT_SYSTEM.md)**.

A short summary:

- **State resolution** (`ResolveMovementState`) picks between WallRunning → AirBorne → Sprinting → Walking → Crouching by priority. JetPacking, Sliding, and Proning are *not* selected by this resolver — they're entered directly by input events, which is the source of most of the state-consistency issues documented below.
- **State transition** (`SetMovementState` → `OnMovementStateChange`) applies per-state `MaxWalkSpeed`, air control, and gravity scale, and runs enter/exit side effects (crouch timelines, wall-run plane constraints, slide velocity/friction).
- **Jetpack thrust** is timer-driven (two ~50 Hz timers for thrust application and fuel drain) while hold-duration/curve sampling is Tick-driven, which creates two independent timing systems feeding the same state.


## License

No license file is currently included in this repository. Treat the project as all-rights-reserved unless a `LICENSE` file is added.
