# FMT-Model-2025b Development Plan Audit

Date: 2026-05-05
Auditor: independent subagent (parallel review)
Scope: architecture v1 + design phase plan + relationships + playbook + rules

## Executive Summary

- The 33-item plan covers Plant/FMS/Controller cleanly and is consistent with INS firmware-ownership, but several **cross-cutting non-functional concerns are missing**: time/clock conventions, init/reset state contract, fault injection, simulation determinism (RNG seeding), generated-code formatting/lint guards, model-reference-vs-library decision, and unit/frame conventions.
- **Architecture §17 risks have partial coverage**: risk 3 (over-abstraction) and risk 7 (variant complexity) lack a dedicated work item with verifiable exit; risk 5 (timing realism) is left to post-design SIH.
- **Open questions partially un-owned**: Q1 (vehicle scope), Q4 (`Control_Out_Bus` as FMS input), Q5 (export automation level), Q6 (`INS_Out_Bus` schema canonicality). Q3 is closed; the rest have no required answer in any item's exit.
- **Sequencing inconsistency**: §4.5 declares `D6 → E1` (strong precedence) but §6 starts E1 in Wave 6 while D6 finalizes in Wave 8. Either E1 splits, or the edge is wrong.
- **Several exit criteria are too vague to objectively verify**: B4, G3, H2, I1, I3, I4, I5.
- **Contract-diff coverage gap**: B4/I3 cover bus/enum/parameter but not `*_EXPORT`/`model_info`/`period` field-order, nor a symbol-presence check for `FMS_PARAM`, `CONTROL_PARAM`, `PLANT_PARAM`, `FMS_init`, `FMS_step`, etc. Architecture §5.1.2/§14.4 demand it; no item enforces it.
- **Methodology risk in playbook §3.1**: Author brief tells subagent to discover upstream via INDEX, which mutates concurrently across parallel Wave authors — breaks "single-conversation independence" claim of §6.
- **Firmware contract spot-check passes**. All buses (`Pilot_Cmd_Bus`, `GCS_Cmd_Bus`, `Auto_Cmd_Bus`, `Mission_Data_Bus`, `INS_Out_Bus`, `Control_Out_Bus`, `FMS_Out_Bus`, `Plant_States_Bus`, `Extended_States_Bus`, `Environment_Info_Bus`, `States_Init_Bus`, IMU/MAG/Baro/GPS/AirSpeed) and exports (`*_PARAM`, `*_EXPORT`, `*_init`, `*_step`) verified in multicopter `lib/` headers. B1–B5 scope is realistic.

## Findings

| ID | Category | Severity | File:Section | Description | Recommended Action |
|---|---|---|---|---|---|
| F-01 | Gap | major | 00-design-plan §4.G G3 | G3 logging exit lists "信号清单/命名约定/可视化目录" but does not require sample-time stamping rules, units metadata, or a manifest update rule. | Tighten G3 exit per Plan Diffs. |
| F-02 | Gap | major | 00-design-plan §4 | No item owns init/reset state contract; arch §15.3 lists it as SIH key check; `*_init` is firmware-visible. | Add A6 "Init/Reset State Contract". |
| F-03 | Gap | major | 00-design-plan §4 | No item owns time/clock convention. `fms_interface_step(uint32_t timestamp)` exists but epoch, wraparound, monotonicity, dt-source unspecified. | Add A7 "Time and Timestamp Conventions". |
| F-04 | Gap | major | 00-design-plan §4 | No item designs fault-injection mechanism. F1/H1 imply but don't require it; arch §17 risk 4 demands MIL-side stress. | Widen F1 noise/bias/delay knobs **or** add H4 fault catalog. |
| F-05 | Gap | major | 00-design-plan §4 | RNG seeding / simulation determinism not specified. ins_stub will need it; H3 baselines need it. | Add seed-management contract to H3 or new I6. |
| F-06 | Gap | major | 00-design-plan §4.I I4 | I4 lists ert.tlc config but does not enforce ban on dynamic memory / recursion / variable-size signals (arch §4.5), nor codegen format/lint. | Tighten I4 exit. |
| F-07 | Gap | major | 00-design-plan §4.A A5 / §4.G G1 | "Model Reference vs Library vs Subsystem" decision per top-level is not exit-required. | Add to A5 exit. |
| F-08 | Gap | minor | 00-design-plan §4.B B3 | Runtime-tunable vs compile-inlined classification not required (arch §14.3 mentions). | Tighten B3 exit. |
| F-09 | Gap | minor | 00-design-plan §4.A | Units / coordinate-frame convention has no item. | Add to A2 or A3 exit. |
| F-10 | Contradiction | major | 01-relationships §4.5 vs §6 | `D6 → E1` strong but Wave 6 starts E1 before D6 (Wave 8). | Split E1 into E1a/E1b; E1b depends on D6. |
| F-11 | Contradiction | minor | 01-relationships §5 vs RULES §3/§6 | Self-check requires upstream "≥ reviewed" but co-seal batches (B1/B2/B3, D3/D4/D6) cannot satisfy literally. | Update RULES §6 to allow co-seal siblings. |
| F-12 | Sequencing | major | 01-relationships §6 Wave 9 | E5 in Wave 9 alongside E3, but `E3 → E5` is strong. | Move E5 to Wave 10 or relax to ⇢. |
| F-13 | Sequencing | minor | 01-relationships §4.7 / §6 | "Frozen-by-Wave" annotation missing for upstreams of late waves (e.g., A4 must stay frozen until Wave 10). | Add "frozen-by" annotation per upstream. |
| F-14 | Sequencing | minor | 01-relationships §4.4/§4.5 | Co-design `D3,D4 ↔ D6` plus `D6 → E1` makes E1 transitively gated by D3/D4 too — conflicts with §6 Wave 6 parallelism. | Resolved with F-10's E1 split. |
| F-15 | Vague exit | major | 00-design-plan §4.B B4 | "diff 工具的输入/输出格式、运行时机、失败响应规则" — no acceptance threshold. | Specify byte-exact field order, numeric-exact enums, etc. |
| F-16 | Vague exit | major | 00-design-plan §4.G G3 | No minimum logged-signal set required. | Require at least one signal per arch §13 module + INS validity + cmd_mask. |
| F-17 | Vague exit | major | 00-design-plan §4.H H2 | Threshold values unspecified; not testable. | Require concrete numeric thresholds per H1 scenario (TBD-ok placeholder). |
| F-18 | Vague exit | minor | 00-design-plan §4.I I1, I3, I4, I5 | All four exits describe deliverables, not pass/fail tests. | Each must include a worked input/output example in the design doc. |
| F-19 | Risk coverage | major | arch v1 §17 risk 3 | Over-abstraction risk has no item with a "minimum vehicle-delta" budget. | Add budget clause to A5 / structural items. |
| F-20 | Risk coverage | major | arch v1 §17 risk 7 | Variant complexity risk; A5 does not bound variant axes for Phase 2. | Tighten A5: enumerate axes, freeze rest until Phase 5. |
| F-21 | Risk coverage | minor | arch v1 §17 risk 5 | Timing realism risk has no design-phase counter (SIL/SIH out of scope). | Require H1 scenarios re-runnable in SIH later. |
| F-22 | Open Q un-owned | major | arch v1 §17 Q1 | Vehicle-scope decision (multicopter-only vs +FW scaffolding) not assigned. | A5 must record. |
| F-23 | Open Q un-owned | major | arch v1 §17 Q4 | `Control_Out_Bus` as FMS input long-term vs transitional — not assigned. | D1 or D6 must answer. |
| F-24 | Open Q un-owned | major | arch v1 §17 Q5 | Export automation level — not assigned. | I5 records manual/semi-auto/auto pick. |
| F-25 | Open Q un-owned | major | arch v1 §17 Q6 | `INS_Out_Bus` schema canonicality unowned; B5 designs mirror flow but does not pick canonical owner or mirror direction. | B5 exit must declare. |
| F-26 | Open Q un-owned | minor | arch v1 §17 Q2 | "Preserve current FMS structurally vs re-author" — not assigned. | D1 must record. |
| F-27 | Consolidation | minor | 00-design-plan §4.B B4 + §4.I I3 | Already a co-design pair (§5); two thin docs likely. | Either merge or enforce single shared review. |
| F-28 | Consolidation | minor | 00-design-plan §4.F F1+F2 | ins_stub functional/structural scope is small; risk of two thin docs. | Consider merging to one F1. |
| F-29 | Split | major | 00-design-plan §4.E E1 | E1 covers loops list **and** cmd_mask trim rules; latter depends on D6. | Split E1a/E1b. Resolves F-10. |
| F-30 | Split | minor | 00-design-plan §4.D D1 | D1 spans 5 sub-areas (modes/sources/shaping/safety/mission). | Tighten exit gate or split mission scope to D1.x. |
| F-31 | Contract risk | major | 00-design-plan §4.B | No item demands `*_EXPORT` / `model_info` / `period` field-order diff. Arch §5.1.2 requires it. | Tighten B4 exit. |
| F-32 | Contract risk | major | 00-design-plan §4.B | No item enforces post-codegen presence check for `FMS_PARAM`, `CONTROL_PARAM`, `PLANT_PARAM`, `FMS_init`, `FMS_step`, `Controller_init`, `Controller_step`, `Plant_init`, `Plant_step`. Symbol-name drift is silent. | Add symbol-presence check to B4/I3. |
| F-33 | Contract risk | minor | 00-design-plan §4.C C3 / §4.D D5 / §4.E E4 | Per-vehicle leaf docs do not require comparing parameter names against firmware `*_PARAM` field names. | Add parameter-name compatibility clause. |
| F-34 | Methodology | major | 02-playbook §3.1 | Author brief: "read INDEX to discover upstream" — INDEX mutates concurrently for parallel Wave authors. Race breaks "single-conversation independence" (§6). | Brief must pass an explicit upstream snapshot at dispatch time. |
| F-35 | Methodology | major | 02-playbook §3.2 + RULES §9 | RULES §9 closing line says "1–6 不达标" but enumerates 7 items; playbook references "1–7". | Fix RULES §9 closing line to "1–7 不达标". |
| F-36 | Methodology | major | 02-playbook §3.3 | Plan Auditor brief constrains report to ≤1500 words, no write path. The current parent task asks for ≤2500 and `_audit/` write path. | Update playbook §3.3 to specify `_audit/<date>-plan-audit.md` and word budget. |
| F-37 | Methodology | minor | 02-playbook §6 | Independence claim partially false until F-34 fixed. | Fixed by F-34. |
| F-38 | INS scope | minor | 00-design-plan §4.F F3 | Consumer-side dependency matrix correctly model-side. No latent INS authoring assumption. | None. |
| F-39 | INS scope | minor | 00-design-plan §4.A A3 | "三个模块" wording confirms INS exclusion. | None. |
| F-40 | INS scope | minor | 00-design-plan §4.B B5 | Correctly framed as mirror flow. | Pair with F-25. |
| F-41 | RULES gap | minor | RULES §3/§6 | Self-check upstream rule conflicts with co-seal batches. | Allow co-seal sibling references. |
| F-42 | RULES gap | minor | RULES §5 | 禁则 does not require recording firmware commit hash for mirrored contract artifacts. Q6/B5 demands it. | Add to §5. |

(Severity = critical | major | minor)

## Recommended Plan Diffs

### `00-design-plan.md`

1. Add **A6 Init/Reset State Contract** (F-02) and **A7 Time/Timestamp Conventions** (F-03) under §4.A.
2. Tighten **B3** exit: each parameter classified runtime-tunable vs compile-inlined (F-08).
3. Tighten **B4** exit: (a) `FMS_EXPORT`/`CONTROL_EXPORT`/`PLANT_EXPORT` field-order diff (F-31), (b) symbol-presence check for all `*_PARAM`, `*_init`, `*_step` (F-32), (c) `period`/`model_info` presence check, (d) byte/numeric exactness thresholds (F-15).
4. Tighten **A5** exit: (a) name model-reference-vs-library decision per top-level (F-07), (b) bound variant axes for Phase 2 (F-20), (c) record vehicle-scope choice (F-22), (d) include over-abstraction budget (F-19).
5. Tighten **D1** exit: record `Control_Out_Bus`-as-FMS-input stance (F-23), preserve-vs-re-author stance (F-26).
6. Tighten **B5** exit: declare canonical `INS_Out_Bus` owner + mirror direction (F-25); record firmware commit hash (F-42).
7. Split **E1** into **E1a** (loops + responsibilities) and **E1b** (cmd_mask trim rules) (F-10/F-29).
8. Tighten **G3** exit: channel naming, sample-time stamps, units metadata, manifest update rule (F-01/F-16).
9. Tighten **H2** exit: numeric thresholds per H1 scenario (F-17).
10. Add **H4 Fault Injection Catalog** (F-04). Or expand F1 noise/bias/delay knob set; pick one.
11. Tighten **H3** exit: seed-management contract (F-05).
12. Tighten **I4** exit: ban dynamic memory/recursion/variable-size signals; codegen format/lint rules (F-06).
13. Tighten **I1**, **I3**, **I5** exits: include worked example in doc (F-18).
14. Add unit/frame convention clause to **A2** or **A3** (F-09).
15. Tighten **C3 / D5 / E4** with parameter-name compatibility clause vs firmware `*_PARAM` field names (F-33).
16. Tighten **I5** to record export-automation level (F-24).
17. Consider merging **F1+F2** or **B4+I3** to avoid thin docs (F-27/F-28).

### `01-design-relationships.md`

1. Replace `D6 → E1` with `D6 → E1b` and `(none) → E1a` (F-10/F-29).
2. Move E5 to Wave 10 or relax `E3 → E5` to `E3 ⇢ E5` (F-12).
3. Add "frozen-by-Wave-N" annotation per upstream-of-late-wave item (F-13).
4. Update §5 co-seal pair list to include `(D3, D4, D6, E1b)` once split.
5. Add A6/A7 dependency edges: `A1 → A6`, `A1 → A7`, `A6 ⇢ B3, C1, D1, E1`, `A7 ⇢ A4, G2, C4, E5`.
6. Add `H4 ⇢ F1`, `H4 → H1` (or co-design).
7. §6 Wave plan: A6/A7 to Wave 2, H4 to Wave 11.

### `02-execution-playbook.md`

1. §3.1 Author brief: replace "Read INDEX to discover upstream" with explicit upstream-path list passed in brief (F-34/F-37).
2. §3.3 Plan Auditor brief: write path = `docs/design/_audit/<date>-plan-audit.md`; word budget aligned with task expectation (F-36).
3. Add §3.4 "Co-seal batch dispatch" describing single-brief vs parallel-with-shared-review for batches (F-11).

### `RULES.md`

1. §6 Self-check: explicit allowance for co-seal batch siblings instead of strict "≥ reviewed" (F-11/F-41).
2. §9 closing line: "1–6 不达标" → "1–7 不达标" (F-35).
3. §5 禁则: add "镜像自 firmware 的契约必须记录 firmware commit hash + 文件路径" (F-42).

### Architecture v1 (only if needed)

1. §17 Q1, Q2, Q4, Q5, Q6: cross-reference owner work item after plan amendments above.
2. §17 risks 3 and 7: cross-reference A5's tightened exit.

## Open Questions for the User

1. Adopt explicit A6 (init/reset) and A7 (time/clock) work items, or fold into stricter A4 exit?
2. F-25/Q6: does the model repo mirror firmware one-way long-term, or is a shared submodule the canonical home? B5 design shape depends on the answer.
3. F-22/Q1: does Phase 2 deliver multicopter only, or fixed-wing scaffolding placeholder? Sets A5 ceiling and downstream abstraction effort.
4. F-29: split E1 (E1a/E1b), or delay all of E1 to Wave 8 alongside D6 (serializes E1 against D-area)?
5. F-04: fault-injection catalog under H (verification) or expanded F1 (stub knobs)?
6. F-06: is "no dynamic memory / no recursion / no variable-size signals" already a hard codegen guard, or does I4 need an explicit MISRA/safety subset?
7. F-34: pass an explicit upstream-snapshot list in each Author brief (eliminates INDEX race) or accept the race window?
8. F-28: split F1/F2 or merge given `ins_stub`'s tiny footprint?

## Methodology Notes

- I read the six required files in full (architecture v1, 00-design-plan, 01-design-relationships, 02-execution-playbook, RULES, INDEX).
- I confirmed firmware-side existence of every bus and exported symbol mentioned in the plan by grepping `Plant_types.h`, `FMS_types.h`, `Controller_types.h`, `Plant.h`, `FMS.h`, `Controller.h` under `FMT-Firmware/src/model/{plant,fms,control}/<vehicle>/lib/`. All bus names, `*_init`, `*_step`, `*_PARAM`, `*_EXPORT` symbols verified present in the multicopter lib/ headers; `Pilot_Cmd_Bus`/`GCS_Cmd_Bus`/`Auto_Cmd_Bus`/`Mission_Data_Bus` present in `FMS_types.h`; `Environment_Info_Bus`/`States_Init_Bus`/`Plant_States_Bus`/`Extended_States_Bus`/IMU/MAG/Baro/GPS/AirSpeed present in `Plant_types.h`.
- I did **not** open `_templates/design-doc-template.md`. RULES findings rely on `RULES.md` text only; if the template diverges from RULES, additional findings may apply.
- I did **not** open per-vehicle headers beyond multicopter (which is the Phase 2 / smallest slice per arch §16). Findings about leaf compatibility (F-33) apply to multicopter only as a pattern for other vehicles.
- I did not open `FMT-Firmware/src/model/ins/cf_ins/lib/INS_types.h`; the `cf_ins` directory exists but I didn't enumerate its `lib/`. Since INS is firmware-owned and `INS_Out_Bus` is already present in `Controller_types.h`/`FMS_types.h`, this is non-blocking.
- "Single-conversation independence" of subagents was assessed against playbook text only; no subagent was actually run.
- All findings are textual analysis of the listed documents; no scripts, codegen, or simulations were executed.
