# FMT-Model-2025b Architecture v1

Date: 2026-05-05
Last updated: 2026-05-05 (revision: INS scoped out of model repo — implemented natively in FMT-Firmware)
Scope: First-pass engineering architecture for the new FMT-Model-2025b repository.
Status: Preliminary architecture only. No implementation in this document.

## 1. Purpose

This document defines the first-pass architecture for FMT-Model-2025b as the model-side companion to FMT-Firmware. The main goal is to establish a clean 2025b-era repository and model decomposition while preserving the firmware contracts that already exist in FMT-Firmware.

The design is grounded in the current firmware-side model interfaces under `FMT-Firmware/src/model/`:
- Plant interface: `plant_interface.h` plus generated `Plant.h/Plant_types.h`
- INS interface: `ins_interface.h` plus `INS.h` — **owned by FMT-Firmware as native C/C++ code, not modeled in this repo**
- FMS interface: `fms_interface.h` plus generated `FMS.h/FMS_types.h`
- Controller interface: `control_interface.h` plus generated `Controller.h/Controller_types.h`

It also reflects the existing architectural intent captured in the project skills:
- FMS as a layered 50 Hz mode-and-command manager
- Controller as a cascaded 200 Hz control stack
- Per-module single-rate design with explicit period budgets
- Deterministic ert.tlc code generation with no dynamic memory

**Scope decision (2026-05-05):** INS is implemented in FMT-Firmware as hand-written C/C++ code and is **out of scope as a primary module of FMT-Model-2025b**. The model repo only:
- consumes `INS_Out_Bus` as an external input contract for FMS/Controller,
- provides a minimal **INS stub/idealization** under the harness layer purely to close the loop for MIL.
This avoids unnecessary churn around an estimator stack that is best maintained in firmware.

This is a repository and module architecture recommendation for the new model repo. It does not change FMT-Firmware behavior by itself.

## 2. Project goals

1. Create a clean Simulink-first repository structure for Plant, FMS, and Controller in FMT-Model-2025b. (INS is **not** a model-side module — it is hand-written in FMT-Firmware.)
2. Preserve firmware integration contracts so generated code can remain a drop-in replacement for the existing firmware model libraries (Plant/FMS/Controller).
3. Treat `INS_Out_Bus` as a stable external contract that the model repo consumes but does not own.
4. Separate shared logic from vehicle-specific logic so multicopter, fixed-wing, VTOL, boat, and car can reuse the same core libraries where appropriate.
5. Make MIL, SIL, HIL/SIH, and firmware integration a planned path rather than an afterthought.
6. Normalize bus, enum, parameter, and codegen management so generated headers remain stable and reviewable.
7. Support a phased progression from smallest executable slice to full vehicle-capable model suite.

## 3. Non-goals

1. Rewriting firmware-side interfaces or topic plumbing in this phase.
2. Full controller/FMS algorithm redesign details for every vehicle in this phase.
3. Immediate support for every historical airframe on day one.
4. Simulink implementation, scripts, or generated code delivery in this phase.
5. **Modeling INS in Simulink at all.** INS lives entirely in FMT-Firmware as native C/C++ code (current PX4 ECL-based estimator stack and any future replacement). The model repo never generates `INS.c/INS.h` and never owns `INS_PARAM`/`INS_EXPORT` style artifacts. The only INS-shaped asset in the model repo is a MIL-only stub/idealization used to close the loop in simulation.

## 4. Design principles

1. Contract-first: preserve current firmware-visible headers, symbol names, field order, enum values, and exported model info structs.
2. Single responsibility: Plant simulates physics, FMS decides vehicle behavior and setpoints, Controller closes control loops and allocates actuators. (INS estimates state but lives in FMT-Firmware, not in this repo.)
3. Shared core, vehicle leaves: shared policy/logic belongs in reusable libraries; vehicle geometry, gains, and allocation belong in leaf packages.
4. Single-rate per module: Plant 1 kHz, Controller 200 Hz, FMS 50 Hz. (INS runs at 100 Hz in firmware; the model repo only consumes `INS_Out_Bus` at that cadence.)
5. Deterministic codegen: ert.tlc, fixed-step discrete, no dynamic memory, no recursion, no variable-size signals.
6. Explicit interfaces: all cross-module communication through named buses and documented rate boundaries.
7. Build for verification: every top-level module must be independently exercisable in MIL before integration.
8. Separate “existing firmware constraints” from “new repo organization recommendations” so reorganization does not accidentally become a contract change.

## 5. Existing firmware constraints vs new 2025b recommendations

### 5.1 Existing firmware contract constraints that must not break

These are observed in the current firmware model integration and should be treated as hard constraints:

1. Module-level integration names (model-repo-owned modules only):
   - Plant exposes `plant_model_info`, `plant_interface_init`, `plant_interface_step`
   - FMS exposes `fms_model_info`, `fms_interface_init`, `fms_interface_step`
   - Controller exposes `control_model_info`, `control_interface_init`, `control_interface_step`
   - INS exposes `ins_model_info`, `ins_interface_init`, `ins_interface_step` — **firmware-owned**, kept here only because Plant/FMS/Controller integration must remain compatible with whatever INS publishes.

2. Generated model entry points and exported structs (model-repo-owned modules only):
   - FMS exports `FMS_init`, `FMS_step`, `FMS_EXPORT`, `FMS_PARAM`
   - Controller exports `Controller_init`, `Controller_step`, `CONTROL_EXPORT`, `CONTROL_PARAM`
   - Plant exports `Plant_init`, `Plant_step`, `PLANT_EXPORT`, `PLANT_PARAM`
   - Export structs contain at least `period` and `model_info[]`
   - **No INS export artifacts are produced by the model repo.** Whatever symbols INS publishes in firmware are firmware's concern.

3. Current module periods implied by skills and generated exports:
   - Plant: 1 ms / 1 kHz (model)
   - Controller: 5 ms / 200 Hz (model)
   - FMS: 20 ms / 50 Hz (model)
   - INS: 10 ms / 100 Hz (firmware-owned wrapper/system integration cadence — model side only consumes `INS_Out_Bus` at this rate)

4. Current FMS input contract must remain compatible:
   - `Pilot_Cmd_Bus`
   - `GCS_Cmd_Bus`
   - `Auto_Cmd_Bus`
   - `Mission_Data_Bus`
   - `INS_Out_Bus`
   - Current interface code also reads `Control_Out_Bus`

5. Current FMS output contract must remain compatible:
   - `FMS_Out_Bus` field order, types, and enum semantics
   - Important fields include rate commands, attitude commands, velocity/acceleration commands, actuator_cmd passthrough space, throttle_cmd, cmd_mask, status/state/ext_state/ctrl_mode/mode/reset/wp fields/home/error

6. Current Controller contract must remain compatible:
   - Inputs: `FMS_Out_Bus`, `INS_Out_Bus`
   - Output: `Control_Out_Bus`

7. Current Plant contract must remain compatible for SIH/HIL paths:
   - Input: `Control_Out_Bus`, `Environment_Info_Bus`, `States_Init_Bus`
   - Outputs: `Plant_States_Bus`, `Extended_States_Bus`, simulated sensor buses such as IMU/MAG/Barometer/GPS/Airspeed

8. Current INS contract must remain compatible **as an external input contract for the model repo**:
   - Output bus: `INS_Out_Bus` (consumed by FMS and Controller)
   - INS itself is implemented in FMT-Firmware as native C/C++ code; the model repo does not author it, does not generate it, and does not export it.
   - Any change to `INS_Out_Bus` field order, types, or semantics must be coordinated with FMT-Firmware first, then mirrored into the model repo's bus definitions.

9. Current enum numeric values are already embedded in generated types, especially in FMS-related state/mode enums. Those values must not drift silently.

10. Current firmware build packaging expects each module to drop generated `.c/.h` into a `lib/` directory next to a thin interface layer.

### 5.2 New 2025b repository organization recommendations

These are recommended for the new repo and may evolve as long as they keep the firmware contracts above stable:

1. One repository-wide source of truth for buses, enums, and parameters.
2. Separate shared libraries from vehicle programs/models.
3. Separate authoring models from generated artifacts and from firmware-export staging.
4. Make scripts reproducible and non-interactive so model creation, verification, and export can run from automation.
5. **Treat INS as firmware-owned.** The model repo holds only an `INS_Out_Bus` schema (mirrored from firmware) and a MIL-only INS stub under the harness layer. There is no `model/ins/` primary module.
6. Standardize directory names, harness placement, and export staging so each model-side module (Plant/FMS/Controller) looks structurally similar.

## 6. Recommended top-level repository tree

Recommended top-level tree for FMT-Model-2025b:

```text
FMT-Model-2025b/
├── docs/
│   └── architecture/
│       └── 2026-05-05-fmt-model-architecture-v1.md
├── model/
│   ├── shared/
│   │   ├── bus/
│   │   │   ├── core/
│   │   │   ├── plant/
│   │   │   ├── ins/                # contract-only: INS_Out_Bus schema mirrored from firmware
│   │   │   ├── fms/
│   │   │   └── controller/
│   │   ├── enum/
│   │   ├── params/
│   │   │   ├── schemas/
│   │   │   └── defaults/
│   │   ├── lib/
│   │   │   ├── math/
│   │   │   ├── filters/
│   │   │   ├── guidance/
│   │   │   ├── control/
│   │   │   ├── safety/
│   │   │   └── utilities/
│   │   └── data/
│   ├── plant/
│   │   ├── shared/
│   │   └── vehicles/
│   │       ├── multicopter/
│   │       ├── fixwing/
│   │       ├── vtol/
│   │       ├── boat/
│   │       └── template/
│   ├── fms/
│   │   ├── shared/
│   │   └── vehicles/
│   │       ├── multicopter/
│   │       ├── fixwing/
│   │       ├── vtol/
│   │       ├── boat/
│   │       └── car/
│   ├── controller/
│   │   ├── shared/
│   │   └── vehicles/
│   │       ├── multicopter/
│   │       ├── fixwing/
│   │       ├── vtol/
│   │       ├── boat/
│   │       ├── car/
│   │       └── submarine/
│   └── harness/
│       ├── ins_stub/               # MIL-only INS idealization; never exported to firmware
│       ├── mil/
│       ├── sil/
│       ├── sih/
│       └── scenarios/
├── scripts/
│   ├── init/
│   ├── build/
│   ├── verify/
│   ├── export/
│   └── dev/
├── config/
│   ├── simulink/
│   ├── codegen/
│   └── vehicles/
├── tests/
│   ├── smoke/
│   ├── contract/
│   └── regression/
└── export/
    ├── firmware/
    │   ├── plant/
    │   ├── fms/
    │   └── controller/           # no ins/ — INS is authored directly in FMT-Firmware
    └── reports/
```

Rationale:
- `model/shared` is the contract and reuse layer.
- `model/<module>/shared` holds module-specific reusable architecture blocks.
- `model/<module>/vehicles/<vehicle>` holds vehicle assembly and leaf specialization.
- `scripts` owns deterministic setup/build/export rather than relying on ad hoc workspace state.
- `export/firmware` becomes the controlled handoff area to firmware packaging.

## 7. Proposed top-level module split

The primary model-side module split for FMT-Model-2025b is **three modules** (Plant/FMS/Controller). INS is firmware-owned and is listed here only as an external contract producer.

1. Plant *(model-owned)*
   - Mission: produce closed-loop truth-state and synthetic sensor environment for simulation and SIH.
   - Owns: rigid-body dynamics, actuator effectiveness in simulation, environment disturbance injection, sensor synthesis.
   - Does not own: estimation, mode logic, control law decisions.

2. FMS *(model-owned)*
   - Mission: convert pilot/GCS/auto/mission intent plus navigation state into vehicle mode, setpoints, and supervisory outputs (`FMS_Out_Bus`).
   - Owns: mode management, command source arbitration, command shaping, mission sequencing, failsafe/safety decisions.
   - Does not own: inner-loop stabilization or actuator mixing beyond any contract-required passthrough fields.

3. Controller *(model-owned)*
   - Mission: convert `FMS_Out_Bus` plus `INS_Out_Bus` into actuator commands (`Control_Out_Bus`).
   - Owns: cascaded control loops, feed-forward, anti-windup, vehicle-specific mixer/allocation.
   - Does not own: mode state machine, geofence policy, mission logic.

— INS *(firmware-owned, not modeled)*
   - Mission: convert raw or simulated sensor information into navigation state estimate (`INS_Out_Bus`).
   - Implementation: hand-written C/C++ inside FMT-Firmware (current PX4 ECL-based stack and any future replacement).
   - Model repo's only relationship: consumes `INS_Out_Bus` as a schema, and may publish a MIL-only stub under `model/harness/ins_stub/` to close the simulation loop.

This split matches the current firmware module boundaries on the model side, while explicitly removing INS as a model-repo responsibility.

## 8. Architectural boundaries by module

### 8.1 Plant boundary

Inputs:
- `Control_Out_Bus`
- `Environment_Info_Bus`
- `States_Init_Bus`

Outputs:
- `Plant_States_Bus`
- `Extended_States_Bus`
- raw/simulated sensor buses: `IMU_Bus`, `MAG_Bus`, `Barometer_Bus`, `GPS_uBlox_Bus`, `AirSpeed_Bus`, optional others

Boundary rule:
- Plant is simulation-only for most workflows. It is not required for firmware flight runtime except SIH/HIL-related integrations.

### 8.2 INS boundary (firmware-owned, contract only)

INS is **not a model-side module**. It is implemented as native C/C++ inside FMT-Firmware. From the model repo's perspective, the only thing that matters is the contract:

Producer (in firmware):
- consumes raw sensor topics and any aiding inputs in firmware's own pipeline
- publishes `INS_Out_Bus` to the rest of the system

Consumer (in this repo):
- FMS and Controller treat `INS_Out_Bus` as an externally produced input
- FMS and Controller must not consume raw IMU/MAG/GPS buses directly
- The model repo holds the canonical `INS_Out_Bus` schema mirrored from firmware; any change in firmware's INS output must be reflected here in lockstep

MIL-only INS stub:
- under `model/harness/ins_stub/`, the repo provides an idealized `INS_Out_Bus` producer driven from `Plant_States_Bus` (truth) plus optional noise/bias injection
- this stub exists solely to close the loop in MIL/SIL scenarios
- the stub is **never** exported to firmware and never participates in a contract diff against firmware-side INS code

### 8.3 FMS boundary

Inputs:
- `Pilot_Cmd_Bus`
- `GCS_Cmd_Bus`
- `Auto_Cmd_Bus`
- `Mission_Data_Bus`
- `INS_Out_Bus`
- existing contract-aware optional `Control_Out_Bus` input if needed to preserve current behavior

Outputs:
- `FMS_Out_Bus`

Boundary rule:
- FMS owns discrete supervision and reference generation only.
- It may shape setpoints, but it must not implement inner-loop PID/ADRC style control.

### 8.4 Controller boundary

Inputs:
- `FMS_Out_Bus`
- `INS_Out_Bus`
- optional `Plant_States_Bus` only in harness/HIL/SIH variants, never as standard flight dependency

Outputs:
- `Control_Out_Bus`

Boundary rule:
- Controller consumes shaped references and estimated state, then produces actuator commands only.
- Vehicle-specific mixer/allocation is the deepest leaf under Controller, not a cross-module service.

## 9. Cross-module buses and dataflow

### 9.1 Primary runtime dataflow

Normal closed-loop dataflow:

```text
Pilot/GCS/Auto/Mission -> FMS -> Controller -> Actuators
                                 ^            |
                                 |            v
                               INS <- Sensors/Plant
```

More explicit engineering dataflow:

```text
Pilot_Cmd_Bus -----\
GCS_Cmd_Bus -------+--> FMS ---------------------> FMS_Out_Bus ----------\
Auto_Cmd_Bus ------/                                                  Controller --> Control_Out_Bus
Mission_Data_Bus --/                                                       ^                |
INS_Out_Bus -------^                                                       |                |
                                                                               INS_Out_Bus <-/

In firmware (production):
Sensors --> [INS firmware code, hand-written C/C++] --> INS_Out_Bus  -> FMS/Controller

In MIL (model repo, simulation only):
Control_Out_Bus --> Plant --> Plant_States_Bus + sensor buses --> [harness/ins_stub] --> INS_Out_Bus
```

### 9.2 Cross-module bus ownership

1. Shared contract buses live in `model/shared/bus`.
2. Firmware-visible bus definitions are treated as externally constrained artifacts.
3. Internal composition buses may exist within a module, but they must not leak into firmware export unintentionally.
4. Internal buses should be module-prefixed to avoid confusing them with contract buses, for example:
   - `FMS_ModeMgr_Bus`
   - `FMS_Safety_Bus`
   - `CTRL_RateLoop_Bus`
   - `PLANT_AeroState_Bus`

### 9.3 Dataflow rules

1. No module may reach into another module’s internal data store.
2. Cross-rate changes occur only in harness or scheduler wiring, not deep within a module.
3. Every contract bus must have one canonical schema source in the model repo.
4. Field order changes in any firmware-visible bus require explicit contract review.

## 10. Shared-vs-vehicle layering

The new repo should use three layers:

### 10.1 Layer A: shared core layer

Purpose:
- common buses, enums, generic library blocks, common verification utilities

Examples:
- deadzones, rate limiters, low-pass filters
- generic mission item decoding helpers
- generic quaternion/rotation utilities
- common safety checks
- common codegen configuration

### 10.2 Layer B: module-shared architecture layer

Purpose:
- reusable module-specific structure shared across vehicles

Examples:
- FMS shared mode manager framework
- FMS shared safety monitor
- Controller shared cascade loop shells
- Plant shared sensor synthesis blocks
- (No INS layer — INS is firmware-owned; the only model-repo INS asset is the harness `ins_stub`, which is a verification fixture and not part of this layer.)

### 10.3 Layer C: vehicle-specific assembly layer

Purpose:
- vehicle topology, gains, mixers, mission constraints, plant coefficients

Examples:
- multicopter allocation matrix
- fixed-wing L1 or coordinated-turn specific leaf logic
- VTOL transition shaper
- boat planar guidance tuning

Rule:
- Vehicle-specific logic should be leaf-level only. Shared mode management, safety policy skeletons, and control cascade structure should stay above the vehicle leaves whenever possible.

## 11. Simulink, script, and codegen layering

The repo should explicitly separate authoring, orchestration, and export layers.

### 11.1 Simulink authoring layer

Contains:
- top models
- linked libraries
- variant definitions
- harness models
- data dictionaries or scripted bus/enum definitions

Responsibility:
- define executable behavior and structure

### 11.2 Script/orchestration layer

Contains:
- init scripts
- model creation/update scripts
- verification runners
- export scripts
- contract comparison scripts

Responsibility:
- reproducibly configure workspace, build variants, run harnesses, and stage outputs

### 11.3 Codegen/export layer

Contains:
- generated code staging outputs
- reports
- firmware-ready file packaging

Responsibility:
- create reviewable handoff artifacts without polluting authoring directories

Rule:
- Generated code must be treated as build output, not as the primary authoring source.
- Export scripts should deliberately place selected outputs into `export/firmware/<module>/<vehicle>/`.

## 12. Recommended module internals

### 12.1 FMS internal architecture

Recommended FMS internal stack, aligned with the stated skill guidance:

1. Command Source Selector
   - pure source arbitration among Pilot/GCS/Auto/Mission-driven requests
   - no mode-transition logic here

2. Mode Manager
   - single source of truth for `VehicleStatus`, `VehicleState`, and control-mode mapping
   - one Stateflow chart for mode/state ownership

3. Safety Monitor
   - evaluates failsafe conditions such as link loss, estimator validity, low battery, geofence, invalid mode preconditions
   - can request mode degradation through the Mode Manager only

4. Mission/Auto Manager
   - mission item sequencing, pause/continue/return/takeoff/land progression
   - separate chart from the Mode Manager

5. Command Shaper
   - convert selected intent into bounded setpoints: angle/rate/velocity/acceleration/yaw-rate/throttle as required by contract
   - own slew/rate/jerk limiting here, not in Controller

6. Output Assembler
   - populate `FMS_Out_Bus`
   - no new logic beyond field assembly and validity mapping

FMS should remain single-rate at 20 ms.

### 12.2 Controller internal architecture

Recommended Controller internal stack, aligned with the stated skill guidance:

1. Setpoint Decode
   - decode `FMS_Out_Bus` masks and select active reference path

2. Position/Outer Guidance Loop as applicable per vehicle
   - only if controller contract really needs it for vehicle type and existing behavior
   - for multicopter, keep reference tracking architecture cascaded and explicit

3. Velocity Loop
   - PI/PID + feed-forward acceleration/thrust terms

4. Attitude Loop
   - quaternion-aware or equivalent attitude error shaping

5. Rate Loop
   - highest-priority hot loop at 200 Hz
   - anti-windup and LPF D-term discipline

6. Mixer/Allocator
   - pure vehicle-specific mapping from force/torque or equivalent command space to actuator channels

7. Output Assembler
   - populate `Control_Out_Bus`

Controller should remain single-rate at 5 ms.

### 12.3 INS — out of scope for the model repo

**Decision (2026-05-05):** INS is implemented in FMT-Firmware as hand-written C/C++ code. The model repo does not author, build, generate, or export an INS module. There is no `model/ins/` directory and no Simulink INS top model.

What lives in the model repo for INS:
1. `INS_Out_Bus` schema under `model/shared/bus/ins/` — mirrored from firmware, treated as an external contract.
2. A MIL-only stub under `model/harness/ins_stub/` whose sole job is to fabricate a plausible `INS_Out_Bus` from `Plant_States_Bus` so MIL/SIL loops can close.

Architectural rationale:
- The current PX4 ECL-derived estimator is non-trivial, sensor-stack-coupled, and benefits from staying in firmware where it can evolve alongside the sensor drivers and topic plumbing.
- Modeling it in Simulink would create churn without delivering matching value: the "model wraps firmware code" pattern from older repos was a maintenance tax.
- Decoupling INS from the model repo lets Plant/FMS/Controller iterate independently of estimator changes, as long as `INS_Out_Bus` stays stable.

What this means in practice:
- INS bugs are fixed in firmware, not in this repo.
- INS tuning parameters are firmware concerns; they do not appear in `model/shared/params/`.
- Any change to `INS_Out_Bus` must originate in firmware, then be mirrored into this repo's bus schema in a coordinated commit.

### 12.4 Plant internal architecture

Recommended Plant structure:

1. Environment/Disturbance Inputs
2. Vehicle Dynamics Core
3. Actuator/Airframe-Specific Dynamics
4. Kinematics/Coordinate Conversion
5. Sensor Synthesis
6. Output Assembly for state and sensor buses

Plant should remain single-rate at 1 ms, with slower sensor publication modeled through output update cadence or harness logic rather than internal multi-rate complexity where possible.

## 13. Timing constraints and scheduling assumptions

The following timing constraints are the architectural baseline:

| Module | Period | Rate | Role | Owner |
|---|---:|---:|---|---|
| Plant | 1 ms | 1000 Hz | simulation hot loop | model repo |
| Controller | 5 ms | 200 Hz | flight control hot loop | model repo |
| FMS | 20 ms | 50 Hz | supervisory logic | model repo |
| INS | 10 ms | 100 Hz | estimator (consumed via `INS_Out_Bus`) | **FMT-Firmware (C/C++)** |

Scheduling assumptions:
1. Each module is internally single-rate.
2. Rate boundaries are explicit at module boundaries or harness wiring.
3. Controller execution budget is most critical; architecture should minimize bus slicing and heavy MATLAB Function use in that path.
4. FMS may tolerate larger logical complexity but must remain deterministic and bounded.
5. Plant and INS latency must be considered in closed-loop MIL/SIL scenarios to avoid hiding integration issues.

Practical implications:
- FMS setpoint shaping must absorb command discontinuities before Controller.
- Controller should not embed supervisory timing assumptions from FMS.
- INS validity flags should be sampled and consumed cleanly at the FMS/Controller boundaries.

## 14. Parameter, bus, and enum management

### 14.1 Bus management

Recommendation:
- Maintain bus definitions from one canonical source under `model/shared/bus/`.
- Generate or derive firmware-facing bus headers from that source, not by ad hoc duplication across module folders.

Rules:
1. Every bus element has explicit type.
2. No `auto`-typed bus elements.
3. Contract buses require comparison against firmware-generated headers during export.
4. Internal buses must be clearly separated from firmware-visible buses.

### 14.2 Enum management

Recommendation:
- Keep all exported enum definitions centralized under `model/shared/enum/`.
- Freeze numeric values for any enum already embedded in firmware contract, especially FMS state/mode/error enums.

Rules:
1. Never replace enum fields with raw integers in authoring models.
2. New enum values require explicit compatibility review.
3. Mode-related enums should be owned by FMS shared definitions, not copied per vehicle.

### 14.3 Parameter management

Recommendation:
- Parameters are managed in typed schemas plus per-vehicle default sets.
- Preserve existing firmware-visible parameter struct names where required: `FMS_PARAM`, `CONTROL_PARAM`, `PLANT_PARAM`.

Rules:
1. Shared schema defines field order and type.
2. Vehicle defaults live separately from schema.
3. Runtime-tunable versus compile-time-inlined parameters must be intentionally categorized.
4. No magic numbers inside models; all operational thresholds and gains route through typed parameter structs.

### 14.4 Contract review gates

Before any export is accepted:
1. Diff field order of generated contract buses versus firmware copies.
2. Diff enum numeric values.
3. Diff exported parameter struct field order if firmware-side parameter linkage depends on it.
4. Confirm exported `period` and `model_info` metadata remain valid.

## 15. MIL, SIL, HIL/SIH, and firmware integration path

Recommended verification progression:

### 15.1 MIL

Purpose:
- algorithm and architecture validation in Simulink

Minimum scope:
- FMS + Controller + Plant closed loop for at least one vehicle
- INS is provided by the harness `ins_stub` (idealized or noise-injected `INS_Out_Bus` derived from `Plant_States_Bus`); the real firmware INS is **not** part of MIL

Key outputs:
- step response
- mode transition correctness
- failsafe behavior
- command shaping quality

### 15.2 SIL

Purpose:
- verify generated C behavior matches model intent (Plant/FMS/Controller only)

Scope:
- per-module SIL first
- then integrated FMS/Controller SIL where practical
- SIL does **not** cover INS — INS has no model-generated counterpart to compare against

Key checks:
- output equivalence within tolerance
- code size and timing profile
- no unexpected helper/runtime bloat

### 15.3 SIH/HIL

Purpose:
- test firmware scheduler/topic integration, including the **real firmware INS** for the first time in the verification chain
- INS appears here because SIH/HIL is where firmware-side code (including the hand-written estimator) actually runs

Scope:
- Plant in SIH path feeding sensor topics
- firmware-side INS (native C/C++) + generated FMS/Controller running under the firmware scheduler
- later hardware IO in HIL

Key checks:
- timestamp assumptions
- topic freshness behavior
- reset/init sequencing
- parameter linkage
- estimator behavior under simulated sensors — this is the first stage where INS contributes its real dynamics, so any contract-level mismatch with `INS_Out_Bus` consumed by FMS/Controller surfaces here

### 15.4 Firmware integration

Purpose:
- drop generated artifacts into firmware-side module `lib/` plus thin interface compatibility (Plant/FMS/Controller only)
- INS continues to live as hand-written firmware code; no integration step needed for it from the model repo

Key checks:
- headers compile without manual edits
- exported symbols match interface code expectations
- module periods and `*_EXPORT` metadata still valid (Plant/FMS/Controller)

Recommended promotion gate:
- MIL pass (with `ins_stub`) -> SIL pass (Plant/FMS/Controller) -> SIH/HIL pass (with real firmware INS) -> firmware branch integration

## 16. Smallest executable slice

The smallest useful executable slice for FMT-Model-2025b should be:

1. One shared bus/enum/parameter foundation (including a mirrored `INS_Out_Bus` schema)
2. One multicopter FMS model at 50 Hz
3. One multicopter Controller model at 200 Hz
4. One minimal multicopter Plant model at 1 kHz for MIL
5. A harness-side `ins_stub` that fabricates `INS_Out_Bus` from `Plant_States_Bus` — purely to close the MIL loop
6. One closed-loop MIL scenario: arm -> takeoff/hold -> manual or position hold -> land/disarm

Why this slice:
- It exercises all three model-side module boundaries (Plant/FMS/Controller) plus the `INS_Out_Bus` consumer contract.
- It validates the repo architecture, not only an isolated algorithm.
- It is small enough to expose contract, timing, and data ownership issues early.
- Multicopter is the best seed because the current firmware already has mature mc_fms, mc_controller, and multicopter plant precedents.

Explicitly excluded from the smallest slice:
- full mission library breadth
- full VTOL transition logic
- **any model-side INS authoring** — INS stays in firmware
- multi-vehicle support

## 17. Risks and open questions

### Top risks

1. Contract drift risk
   - Reorganizing buses/enums/parameters in the new repo could silently change generated field order, enum values, or exported struct naming, breaking firmware drop-in compatibility.

2. `INS_Out_Bus` contract drift risk *(replaces previous "INS architecture mismatch risk")*
   - Because INS now lives entirely in firmware, the model repo only mirrors `INS_Out_Bus`. If firmware changes the bus and the model repo is not updated in lockstep, MIL will silently diverge from real flight behavior and SIH will be the first place the mismatch surfaces.
   - Mitigation: treat `INS_Out_Bus` as a versioned contract; a contract-diff script must compare the model repo's mirrored definition against firmware on every export.

3. Shared-vs-vehicle over-abstraction risk
   - Trying to unify all vehicle classes too early could produce generic libraries that are elegant on paper but awkward for multicopter/fixed-wing/VTOL specifics, slowing first delivery.

4. INS-stub realism risk *(new)*
   - A too-idealized `ins_stub` in MIL may hide control or FMS issues that real estimator noise/delay would expose. MIL passes that depend on perfect state will mislead engineers.
   - Mitigation: provide configurable noise/bias/delay knobs in the stub; require at least one MIL scenario to run with realistic disturbance settings before promoting to SIH.

### Additional risks

5. Timing realism risk
   - MIL may look stable while scheduler/topic/timestamp behavior in firmware or SIH reveals coupling bugs.

6. Parameter governance risk
   - Untyped or duplicated parameter definitions across scripts and models can cause hard-to-review tuning divergence.

7. Variant complexity risk
   - Too many variant controls too early can make codegen/debug/export brittle.

### Open questions

1. Should the first 2025b delivery target multicopter only, or multicopter plus fixed-wing shared scaffolding?
2. How much of the current FMS behavior should be preserved structurally versus re-authored cleanly behind the same output contract?
3. ~~Will INS in 2025b remain a wrapper package around existing estimator code, or is there a roadmap for partial model-side ownership?~~ **Resolved (2026-05-05): INS stays in FMT-Firmware as native C/C++ code; the model repo only mirrors `INS_Out_Bus` and provides a MIL stub.**
4. Should `Control_Out_Bus` remain an FMS input long-term, or is that only a transitional compatibility path?
5. What level of export automation is desired for copying or syncing generated artifacts into FMT-Firmware?
6. Where does `INS_Out_Bus` schema versioning live — in the model repo, in firmware, or in a shared submodule? Need a single source of truth and a diff workflow.

## 18. Phased progression

This architecture should be executed in phases aligned with systems-engineering style progressive design maturity.

### Phase 0: Architecture baseline

Deliverables:
- this architecture document
- agreed repo tree
- agreed contract inventory
- agreed first vehicle target

Exit criteria:
- architecture review accepted
- hard contracts frozen for phase 1

### Phase 1: Shared foundation

Deliverables:
- shared bus/enum/parameter source of truth (including a mirrored `INS_Out_Bus` schema)
- init/build/export script skeleton
- codegen configuration baseline
- contract diff tooling — including a dedicated `INS_Out_Bus` consumer-side diff against FMT-Firmware

Exit criteria:
- one no-op or stub export can reproduce firmware-compatible headers/artifacts layout for Plant/FMS/Controller
- `INS_Out_Bus` mirror diff runs clean against current firmware definition

### Phase 2: Smallest executable multicopter slice

Deliverables:
- multicopter FMS architecture shell
- multicopter Controller architecture shell
- minimal Plant for MIL
- harness `ins_stub` producing a configurable `INS_Out_Bus` from `Plant_States_Bus`
- one closed-loop MIL scenario

Exit criteria:
- arm/hold/basic reference-tracking closed loop runs in MIL using `ins_stub`
- generated FMS/Controller headers pass contract diff review

### Phase 3: Integration hardening

Deliverables:
- SIL runs (Plant/FMS/Controller only — INS has no SIL counterpart)
- SIH/HIL wiring against the **real firmware INS** (this is the first time real INS dynamics enter the verification chain)
- export-to-firmware packaging (Plant/FMS/Controller)
- timing/profile review

Exit criteria:
- generated artifacts integrate with firmware without manual contract fixes
- closed-loop SIH with real firmware INS passes basic functional and timing checks
- `INS_Out_Bus` consumer code in FMS/Controller behaves consistently between MIL (stub) and SIH (real INS) within agreed tolerance

### Phase 4: Feature completion for primary vehicle

Deliverables:
- richer mode set
- mission/auto branch completion
- safety/failsafe coverage
- controller performance tuning

Exit criteria:
- primary vehicle architecture is feature-complete enough for broader engineering iteration

### Phase 5: Vehicle expansion

Deliverables:
- fixed-wing/VTOL/boat/car reuse rollout based on shared foundations
- explicit leaf specializations and harness reuse

Exit criteria:
- second vehicle class reuses shared architecture with limited duplication

## 19. Recommended first implementation target after this document

The first implementation target should be a multicopter-only vertical slice because it best fits the current firmware maturity and minimizes ambiguity:
- Plant: multicopter
- FMS: multicopter architecture shell
- Controller: multicopter cascaded shell
- INS: harness `ins_stub` only — real INS continues to live in FMT-Firmware
- Harness: one MIL end-to-end scenario

This keeps the problem bounded while still validating the core repo architecture.

## 20. Summary of architectural recommendation

1. Keep the current firmware module contracts stable.
2. Rebuild the model repo around shared contract assets, shared module architecture, and vehicle leaf assemblies.
3. **INS is firmware-owned C/C++, not a model-side module.** The model repo only mirrors `INS_Out_Bus` and provides a MIL stub.
4. Use multicopter as the seed vehicle.
5. Prove the architecture through the smallest executable closed-loop slice (Plant + FMS + Controller + `ins_stub`) before broadening vehicle support.

That approach minimizes contract risk, avoids premature over-generalization, removes an unnecessary model-side maintenance tax around the estimator, and creates a credible path from MIL to firmware integration.