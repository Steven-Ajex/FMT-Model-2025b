---
work_item: D3
title: FMS Mode Manager 详细设计
upstream: [架构v1, A1, A2, A3, A4, A5, A6, A7, A8, B1, B2, B3, D1, D2, D4, D6, E1]
contract_impact: no
status: reviewed
authored_at: 2026-05-08
last_reviewed_at: 2026-05-08
reviewer_verdict: pass
---

# D3 FMS Mode Manager 详细设计

## 1. 目的

把 [D2 §4.3.2](D2-fms-structural.md) 已锁定的 Mode Manager 子模块 — 即架构 v1 §12.1.2 的"FMS 内部唯一 Stateflow chart"— 的内部状态层级、转换、事件、守卫、以及 mode↔`VehicleStatus`/`VehicleState`/`VehicleExtState`/`FlightMode`/`CtrlMode`/`FailsafeState` 映射、cmd_mask 位组合 emission 全部冻结,作为 [D4 Command Shaper](D4-command-shaper.md) 的 routing 输入与 [D6 cmd_mask 语义](D6-fms-controller-interface.md) §4.6.1 D3 hand-off 表的来源,并通过 `FMS_Out_Bus.cmd_mask` + `ctrl_mode` 间接给 [E1 Controller 功能](../E-controller/E1-controller-functional.md) 提供环路裁剪输入。

## 2. 范围

**在范围:**

- Stateflow chart 的状态层级(super-state / OR / AND-state)
- 12 主 mode(M-01..M-12,per [D1 §4.1.1](D1-fms-functional.md))+ 4 failsafe sub-mode(FS-01..FS-04,per D1 §4.1.1 表 2)的 Stateflow 状态布置
- 每个状态到 [B2 §4.4.1..4.4.7 / §4.4.10](../B-contracts/B2-enum-inventory.md) `VehicleStatus` / `VehicleState` / `VehicleExtState` / `FlightMode` / `CtrlMode` / `FailsafeState` / `ErrorCode` 的映射(D3 主退出)
- 转换(source → target / 触发 event / 守卫 guard / action / 优先级)
- 事件 inventory(命名 + 来源 bus 字段)
- 守卫表达式(declarative,引用 [B3](../B-contracts/B3-parameter-schema.md) FMS_PARAM 字段名;不锁数值)
- 每个 ARMED 状态在 `during` / `entry` 上 emit 的 `cmd_mask` 位组合(契约 — 满足 [D6 §4.4.1](D6-fms-controller-interface.md) 合法 truth-table + §4.6.1 D3 hand-off 表)
- Reset / Init 行为(per [A6](../A-architecture/A6-init-reset-contract.md) + D1 §4.1.3)
- Failsafe overlay 结构(D3 选 AND-state overlay 的论证)
- Hand-off 给 D4 / D6 / E1 的具体内容

**不在范围(由其他工作项处理):**

- `cmd_mask` 每位的**控制律语义**(置位时 Controller 做什么)— 由 [D6 §4.2](D6-fms-controller-interface.md) 锁(co-seal)
- Setpoint 字段**数值生成公式**(rate/jerk-limited 数学、起飞/降落 profile 数值)— 由 [D4](D4-command-shaper.md) 锁(co-seal)
- Controller 各环 **trim** 算法 — 由 [E1 §4.2](../E-controller/E1-controller-functional.md) 锁(co-seal)
- 多旋翼专属 entry guard 数值默认 / takeoff/landing profile 常量 — 由 [D5 多旋翼 leaf](D5-multicopter-leaf.md) 锁(Wave 9)
- bus / enum / parameter 字段级 schema — 由 [B1](../B-contracts/B1-bus-inventory.md) / [B2](../B-contracts/B2-enum-inventory.md) / [B3](../B-contracts/B3-parameter-schema.md) 拥有
- Mission 内部 sequencer(wp 推进 / pause / resume) — 由 D2 §4.3.4 Mission Manager 拥有(本文件只处理 Mode Manager 与 Mission Manager 之间通过 `FMS_Mission_to_ModeMgr_Bus` 的边界事件)
- Safety Monitor 内部触发 debounce 数值 — 由 D2 §4.3.3 拥有(本文件只处理 Safety 通过 `FMS_Safety_to_ModeMgr_Bus` 给 D3 的降级请求)
- Source Selector 内部仲裁(本文件 Stateflow 直接消费 `FMS_SrcSel_Bus.{pilot_mode_request, gcs_mode_request, auto_mode_request}` 等已选好的字段;源仲裁见 D2 §4.3.1 + D1 §4.2)

## 3. 依赖

| 上游 | 引用位置 | 用途 |
|---|---|---|
| 架构 v1 §12.1.2 | [架构 v1](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | "Mode Manager 是单一 Stateflow chart;`VehicleStatus` / `VehicleState` / control-mode mapping 的 single source of truth" |
| A2 命名与单位约定 | [A2](../A-architecture/A2-naming-conventions.md) | Stateflow 状态名命名 / chart 块名 / 变量前缀 |
| A3 §4.4(FMS 输入)/ §4.4.4(INS validity)/ §4.4.6.2(状态 enum 字段)| [A3](../A-architecture/A3-module-boundaries.md) | Mode Manager 消费的 INS validity 位 / pilot mode_switch / GCS cmd_type / 输出的状态 enum 字段集 |
| A4 §4.4 RB-04/05/06 | [A4](../A-architecture/A4-rate-boundaries.md) | chart 内部全 20 ms;输入端走 RB-04(INS) / RB-06(外部),不依赖跨速率边沿事件 |
| A5 变体策略 | [A5](../A-architecture/A5-variant-strategy.md) | chart 拓扑 shared,leaf 注入仅通过 PARAM(D5)— D3 chart 不切 Variant Subsystem |
| A6 §4.4.2 reset 类别表 / §4.5.2 global vs local reset / §4.5.3 INS readiness gate / §4.7.1 INS gate sequence | [A6](../A-architecture/A6-init-reset-contract.md) | chart entry state(M-01) / 状态 enum reset 值(per 类别 PARAM/HARDCODED/ZERO/PRESERVED)/ INS readiness gate 守卫 |
| A7 时间约定 | [A7](../A-architecture/A7-time-conventions.md) | dwell / debounce 计时器消费 step interface timestamp(单位 ms;固定步长派生 dt = 20 ms) |
| A8 §4.3.2 `FMS_Mode_Manager_Framework` | [A8](../A-architecture/A8-shared-library-roster.md) | chart 骨架 Layer B FMS shared shell |
| B1 §4.2(FMS 输入 bus) / §4.7(`FMS_Out_Bus` 状态字段)| [B1](../B-contracts/B1-bus-inventory.md) | 字段名 / 类型;字段级 schema 由 B1 拥有 |
| B2 §4.4.1..§4.4.10 + §4.4.13 + §4.4.14 | [B2](../B-contracts/B2-enum-inventory.md) | `VehicleStatus` / `VehicleState` / `VehicleExtState` / `PilotMode` / `FlightMode` / `CtrlMode` / `FailsafeState` / `MissionState` / `ErrorCode` / `GCS_CmdType` / `CmdMaskBit` 的成员清单 + 数值;D3 不复述 |
| B3 §4.4.2 FMS_PARAM | [B3](../B-contracts/B3-parameter-schema.md) | 守卫表达式引用的 PARAM 名(`arm_throttle_thr_n01` / `arm_safety_check_mask` / `link_loss_timeout_s` / `mode_switch_debounce_s` / `failsafe_recovery_dwell_s` / `ins_unready_timeout_s` 等);数值不锁 |
| D1 §4.1 / §4.2.2 / §4.4(failsafe 触发表)/ §4.5(mission 范围) | [D1](D1-fms-functional.md) | mode 集合(12 主 + 4 failsafe)/ 仲裁优先级 / 11 触发分类 / mission 子集;D3 把这些功能层裁决落到 Stateflow 转换图上 |
| D2 §4.3.2(Mode Manager 边界)/ §4.4(D3 hand-off)/ §4.2.1(`FMS_ModeMgr_to_Shape_Bus` / `FMS_ModeMgr_to_Asm_Bus` 字段类别)| [D2](D2-fms-structural.md) | Stateflow 输入 / 输出 bus 字段类别;Mode Manager 是 chart 形式;reset entry state |
| **(co-seal Wave 8 batch sibling)** D4 Command Shaper | [D4](D4-command-shaper.md) | D3 通过 `FMS_ModeMgr_to_Shape_Bus.{active_mode, ctrl_mode_hint, route_*, enable_*}` 把 routing 指令喂给 D4;D4 据此实现整形公式 |
| **(co-seal Wave 8 batch sibling)** D6 FMS↔Controller 接口 | [D6](D6-fms-controller-interface.md) | §4.2 cmd_mask 每位语义 / §4.4 互斥规则 / §4.5 INS-invalid emission 政策 / §4.6.1 D3 hand-off 表(D3 §4.6 表满足 D6 §4.6.1 列头与契约形状) |
| **(co-seal Wave 8 batch sibling)** E1 Controller 功能 | [E1](../E-controller/E1-controller-functional.md) | D3 emit 的 cmd_mask + ctrl_mode 是 E1 §4.2 trim 算法的输入;E1 反向不约束 D3 chart 拓扑 |
| FMT-Firmware @ `<pending hash>` | `FMT-Firmware/src/model/fms/<vehicle>/lib/FMS_types.h` | enum 数值 mirror 源(B2 拥有镜像;D3 不直读 firmware,以 B2 / B3 已锁字段为准)— D1 §4.8 OQ2 已锁 "macro 保留 + leaf 重写",chart 拓扑允许与 firmware 当前 Stateflow diverge |

环境注:FMT-Firmware 在本设计阶段**未挂载**;本文件以"D1 / D2 已锁功能与边界 + B2 enum 已锁数值 + B3 PARAM 已锁字段名 + D6 cmd_mask 语义(同 batch)"派生 chart 拓扑,不直接读 firmware 头文件。同 batch 兄弟 D4 / D6 / E1 在 draft 状态相互引用 — 已在本表标 "(co-seal Wave 8 batch sibling)"(per [RULES §6](../RULES.md))。

## 4. 设计内容

### 4.1 Stateflow chart 层级

#### 4.1.1 顶层结构(super-states + AND-state failsafe overlay)

Mode Manager 是**单一 Stateflow chart**(per 架构 v1 §12.1.2 + [D2 §4.3.2](D2-fms-structural.md));chart 顶层并行两条 AND-state 通道:

```text
+============================================================================+
|  CHART: FMS_Mode_Manager  (Stateflow, 20 ms, FMS shared per A8 §4.3.2)     |
|                                                                            |
|  +======================== AND-state #1: ARM_DOMAIN =====================+ |
|  |                                                                        | |
|  |  OR-state DISARMED (super)                  OR-state ARMED (super)     | |
|  |  +----------------------+                   +----------------------+   | |
|  |  | OR-state INIT        |  arm event        | OR-state MAIN_MODE   |   | |
|  |  |  (first-boot sub)    | ----------->      |  +-----------------+ |   | |
|  |  +----------------------+                   |  | M-02 MANUAL     | |   | |
|  |  | OR-state STANDBY     |                   |  +-----------------+ |   | |
|  |  | (steady disarmed)    |                   |  | M-03 STABILIZE  | |   | |
|  |  +----------------------+                   |  +-----------------+ |   | |
|  |                                  <--------- |  | M-04 ALTHOLD    | |   | |
|  |                                  disarm     |  +-----------------+ |   | |
|  |                                              |  | M-05 POSHOLD    | |   | |
|  |                                              |  +-----------------+ |   | |
|  |                                              |  | M-06 TAKEOFF    | |   | |
|  |                                              |  +-----------------+ |   | |
|  |                                              |  | M-07 LAND       | |   | |
|  |                                              |  +-----------------+ |   | |
|  |                                              |  | M-08 RTL        | |   | |
|  |                                              |  +-----------------+ |   | |
|  |                                              |  | M-09 LOITER     | |   | |
|  |                                              |  +-----------------+ |   | |
|  |                                              |  | M-10 MISSION    | |   | |
|  |                                              |  +-----------------+ |   | |
|  |                                              |  | M-11 OFFBOARD   | |   | |
|  |                                              |  +-----------------+ |   | |
|  |                                              |  | M-12 ACRO       | |   | |
|  |                                              |  +-----------------+ |   | |
|  |                                              +----------------------+   | |
|  +========================================================================+ |
|                                                                            |
|  +======================== AND-state #2: FAILSAFE_OVERLAY ===============+ |
|  |                                                                        | |
|  |  OR-state NORMAL (default)                                             | |
|  |  +-----+   safety event                                                | |
|  |  +-----+ -------------+                                                | |
|  |              |                                                         | |
|  |              v                                                         | |
|  |  +----------------------+                                              | |
|  |  | FS-01 RTL (failsafe) |  recovers via D1 §4.4.4 + dwell              | |
|  |  +----------------------+                                              | |
|  |  | FS-02 LAND_NOW       |  forces touchdown then DISARM               | |
|  |  +----------------------+                                              | |
|  |  | FS-03 HOVER_HOLD     |  POSHOLD overlay w/ failsafe_state set       | |
|  |  +----------------------+                                              | |
|  |  | FS-04 DISARM         |  forces DISARMED + LOCKDOWN                 | |
|  |  +----------------------+                                              | |
|  +========================================================================+ |
+============================================================================+
```

#### 4.1.2 关键拓扑决策

| 决策 | 选择 | 理由 |
|---|---|---|
| **DISARMED / ARMED 是 OR-state 互斥**(同一时刻仅一条活跃)| OR-state(super-state) | 一架飞行器同时 DISARMED + ARMED 在物理上不可能;不需要并行;`VehicleStatus` 的 `STATUS_DISARM` / `STATUS_STANDBY` / `STATUS_ARM` 三态在此层一一对应 |
| **12 主 mode 全部 = ARMED 内的 OR-state**(M-02..M-12 + ARMED 内的 IDLE-equiv)| OR-state,**单层** flat | mode 之间互斥(D1 §4.1.4 类别矩阵);单层 flat 比类别分组(AUTO / MANUAL / FAILSAFE)更简单且与 D1 一一对应;若将来 D5 leaf 想分组可加 super-state container 不破坏拓扑 |
| **DISARMED 内含 INIT + STANDBY**(两个 OR-state) | OR-state 嵌套 | INIT 只在首次启动出现(等待 INS_Status>=READY);STANDBY 是稳态待机;两态分开便于 mode-at-reset 行为定义(per [A6 §4.5.3](../A-architecture/A6-init-reset-contract.md))|
| **4 failsafe sub-mode = AND-state overlay**(parallel) | AND-state 并行 | 见 §4.8 详细论证;关键点:failsafe 可在**任何** ARMED main-mode 上触发,且失效解除后能让 main mode 从被 SUSPEND 的位置 resume(若 D1 §4.4.4 标"可恢复");若设为 OR-state(与 main mode mutex),则解除后必须重新走 entry,会丢失 main mode 的内部 dwell / phase 状态 |
| **chart 不分多个 chart**(架构 v1 §12.1.2 锁定 = ONE chart)| 单一 chart | 架构 v1 §12.1.2 + D2 §4.3.2 已锁;Mission Manager / Safety Monitor 是独立 Simulink 块(非 chart),通过 `FMS_Mission_to_ModeMgr_Bus` / `FMS_Safety_to_ModeMgr_Bus` 进入本 chart |

#### 4.1.3 Phase 2 多旋翼 chart 实例化范围

per [D1 §4.1.2](D1-fms-functional.md) + [A5 Phase 2 = 多旋翼唯一 leaf](../A-architecture/A5-variant-strategy.md):

- **ARM_DOMAIN AND-state**:DISARMED.{INIT, STANDBY} + ARMED.{M-01..M-12} 全启用(M-12 ACRO 在 PilotMode 缺 `PMODE_ACRO` 时只能由 GCS `CMDTYPE_MODE_CHANGE` 触发,详见 §4.3.2 注;D5 leaf 决定是否启用)
- **FAILSAFE_OVERLAY AND-state**:NORMAL + FS-01..FS-04 全启用
- **chart 总状态数**:2(ARM_DOMAIN super)+ 2(DISARMED inner OR)+ 12(ARMED inner OR)+ 1(FAILSAFE_OVERLAY super,若计 super)+ 5(NORMAL + FS-01..FS-04)= 22 个 leaf 状态(可直接落到 Stateflow 编辑器)

> 注:M-01 在 D1 §4.1.1 中称为 "DISARMED / IDLE",在 chart 中分裂为 DISARMED.INIT(首次启动)+ DISARMED.STANDBY(稳态);ARMED 进入后 chart 不再拥有 "M-01" 状态(M-01 等价 = DISARMED 整体)。这里把 D1 工作名 M-01 映射到 chart 的 `DISARMED` super-state。

### 4.2 状态 ↔ B2 enum 映射表(D3 主退出条件)

每个 chart leaf 状态在 `during` 上输出以下 enum 字段值,经 `FMS_ModeMgr_to_Asm_Bus`(per [D2 §4.2.1](D2-fms-structural.md))进入 `FMS_Out_Bus`(by Output Assembler)。**enum 数值** 由 [B2 §4.4.1..7 / §4.4.10](../B-contracts/B2-enum-inventory.md) 锁;本表只锁 (state, enum 成员名) 的对应。`error_code` 在无错时 = `ERR_NONE`,故 §4.5 / §4.8 中只对触发态列出非 NONE 的 ErrorCode。

| Chart 状态 | `VehicleStatus` (B2 §4.4.1) | `VehicleState` (B2 §4.4.2) | `VehicleExtState` (B2 §4.4.3) | `FlightMode` (B2 §4.4.5) | `CtrlMode` (B2 §4.4.6) | `FailsafeState` (B2 §4.4.7) | 备注 |
|---|---|---|---|---|---|---|---|
| **DISARMED.INIT** | `STATUS_DISARM` | `STATE_DISARM` | `EXT_STATE_LANDED` | `MODE_NONE` | `CMODE_DISARMED` | `FAIL_NONE` | 首启;等 INS readiness gate(A6 §4.5.3)|
| **DISARMED.STANDBY** | `STATUS_STANDBY` | `STATE_STANDBY` | `EXT_STATE_LANDED` | `MODE_NONE` | `CMODE_DISARMED` | `FAIL_NONE` | INS ready 后稳态;等 arm 命令 |
| **ARMED.M-02 MANUAL** | `STATUS_ARM` | `STATE_GROUND`(throttle 未起飞)/ `STATE_INAIR`(已起飞;由 takeoff_detected 翻面)| `EXT_STATE_LANDED`(地面)/ `EXT_STATE_MOVING`(空中)| `MODE_MANUAL` | `CMODE_MANUAL_RATE` | `FAIL_NONE` | rate 直传 + throttle 直传 |
| **ARMED.M-03 STABILIZE** | `STATUS_ARM` | `STATE_GROUND` / `STATE_INAIR` | `EXT_STATE_LANDED` / `EXT_STATE_MOVING` | `MODE_STABILIZE` | `CMODE_MANUAL_ANGLE` | `FAIL_NONE` | attitude + throttle 直传 |
| **ARMED.M-04 ALTHOLD** | `STATUS_ARM` | `STATE_INAIR`(ALTHOLD 默认假设已起飞;若地面 entry 拒绝,见 §4.3 ENTRY-GUARD)| `EXT_STATE_HOLDING`(stick 中位)/ `EXT_STATE_MOVING`(stick 偏)| `MODE_ALTHOLD` | `CMODE_AUTO_VEL` | `FAIL_NONE` | z-vel 闭环 + 水平 attitude 手动 |
| **ARMED.M-05 POSHOLD** | `STATUS_ARM` | `STATE_INAIR` | `EXT_STATE_HOLDING` / `EXT_STATE_MOVING` | `MODE_POSHOLD` | `CMODE_AUTO_POS` | `FAIL_NONE` | 全位置闭环 |
| **ARMED.M-06 TAKEOFF** | `STATUS_ARM` | `STATE_INAIR`(takeoff sequence 持续期间)| `EXT_STATE_TAKING_OFF` | `MODE_NONE`(D1 §4.1.1 "MODE_NONE 派生 + ext_state=TAKING_OFF";若 firmware 后续增 `MODE_TAKEOFF`,本行 §10 changelog 同步)| `CMODE_AUTO_POS` | `FAIL_NONE` | takeoff profile 由 D4 + Mission Manager 联动 |
| **ARMED.M-07 LAND** | `STATUS_ARM` | `STATE_INAIR`(下降中)→ `STATE_GROUND`(touchdown)| `EXT_STATE_LANDING` | `MODE_LAND` | `CMODE_AUTO_VEL` | `FAIL_NONE` | descent profile;touchdown 后转 DISARM |
| **ARMED.M-08 RTL** | `STATUS_ARM` | `STATE_INAIR` | `EXT_STATE_RETURNING` | `MODE_RTL` | `CMODE_AUTO_POS` | `FAIL_NONE` | 三段:climb → cruise → land(转 M-07)|
| **ARMED.M-09 LOITER** | `STATUS_ARM` | `STATE_INAIR` | `EXT_STATE_HOLDING` | `MODE_POSHOLD`(D1 §4.1.1 "复用 MODE_POSHOLD";若 firmware 增 `MODE_LOITER`,本行 §10 changelog)| `CMODE_AUTO_POS` | `FAIL_NONE` | wp 处停留 |
| **ARMED.M-10 MISSION** | `STATUS_ARM` | `STATE_INAIR` | `EXT_STATE_MOVING`(wp 间巡航)/ `EXT_STATE_HOLDING`(wp loiter 内嵌)| `MODE_MISSION` | `CMODE_AUTO_POS` | `FAIL_NONE` | wp 推进 |
| **ARMED.M-11 OFFBOARD** | `STATUS_ARM` | `STATE_INAIR` | `EXT_STATE_MOVING` | `MODE_OFFBOARD` | per `FMS_SrcSel_Bus.auto_cmd_mask_request`(D6 §4.6.1 enforces 合法过滤;**默认** `CMODE_AUTO_POS` 当 Auto target 为 pos)| `FAIL_NONE` | external-driven |
| **ARMED.M-12 ACRO** | `STATUS_ARM` | `STATE_GROUND` / `STATE_INAIR` | `EXT_STATE_LANDED` / `EXT_STATE_MOVING` | `MODE_ACRO` | `CMODE_MANUAL_RATE` | `FAIL_NONE` | rate-stick 直传 |
| **FAILSAFE_OVERLAY.NORMAL** | (透明 — main mode 写)| (透明)| (透明)| (透明)| (透明)| `FAIL_NONE` | overlay 默认态;不写 status/state/mode/ctrl_mode/ext_state(由 main mode 写)|
| **FAILSAFE_OVERLAY.FS-01 RTL_FAILSAFE** | `STATUS_ARM`(继续飞)| `STATE_INAIR` | `EXT_STATE_RETURNING` | `MODE_RTL`(overlay 写 mode = RTL,SUSPEND main)| `CMODE_AUTO_POS` | per 触发源:`FAIL_RC_LOSS` / `FAIL_GCS_LOSS` / `FAIL_LOW_BATTERY` / `FAIL_GEOFENCE` | per D1 §4.4.1 TR-01/02/05/07 → FS-01 |
| **FAILSAFE_OVERLAY.FS-02 LAND_NOW** | `STATUS_ARM` | `STATE_INAIR` → `STATE_GROUND`(touchdown 后)| `EXT_STATE_LANDING` | `MODE_LAND` | `CMODE_AUTO_VEL` | `FAIL_LOW_BATTERY`(critical 子级) | per D1 §4.4.1 TR-06 / TR-11 → FS-02;LAND profile 同 M-07 |
| **FAILSAFE_OVERLAY.FS-03 HOVER_HOLD** | `STATUS_ARM` | `STATE_INAIR` | `EXT_STATE_HOLDING` | `MODE_POSHOLD` | `CMODE_AUTO_POS` | `FAIL_GCS_LOSS` / `FAIL_OFFBOARD_LOSS` | per D1 §4.4.1 TR-02 / TR-09(action=`FA_HOLD`)|
| **FAILSAFE_OVERLAY.FS-04 DISARM_FORCED** | `STATUS_DISARM` | `STATE_LOCKDOWN` | `EXT_STATE_LANDED` | `MODE_NONE` | `CMODE_DISARMED` | `FAIL_INS_INVALID`(或触发源对应)| per D1 §4.4.1 TR-03 / TR-10 / TR-11(motor)→ FS-04;forced disarm,不可恢复(D1 §4.4.4)|

**FS-* overlay 的"覆盖语义"**:per §4.8,FAILSAFE_OVERLAY AND-state 在 NORMAL 时**透明**(让 ARM_DOMAIN main mode 写所有 enum + setpoint routing);进入 FS-01..FS-04 时**接管** mode / ctrl_mode / failsafe_state / error_code 字段,且**接管** `FMS_ModeMgr_to_Shape_Bus.route_*` 路由指令(让 D4 按 failsafe target mode 整形 setpoint)。同时 ARM_DOMAIN 中的 main mode 状态 **SUSPEND**(保留内部计时器 / dwell / wp_index for resume)— 实现细节见 §4.8.2。

**关键不变量**:
1. 每个 chart leaf 状态(去除 NORMAL overlay)对 6 个 enum 字段都有**确定**的成员名映射(无 "TBD" 行);D3 退出条件之一。
2. `error_code`:仅在**触发**事件命中那一帧由 chart entry action 写非 NONE 值;之后保持直到下一次 mode 转换(见 §4.3 行为约定)或 `local reset` 清零(per D1 §5 风险条目 + §4.7)。
3. `(VehicleStatus, VehicleState)` 二元组在 chart 中始终成对写(per [B2 §4.4.2 备注](../B-contracts/B2-enum-inventory.md));D3 不允许某 leaf 状态只写其一。

### 4.3 转换(Transitions)

#### 4.3.1 转换表语义约定

每条转换以 `[priority] source → target [event] guard / action` 表示;同源同 priority 的转换间冲突由 Stateflow 默认 deterministic 顺序解决(见 §4.3.5)。**Action** 列在状态机层只描述**控制流副作用**(如清计时器 / 设置 latched 字段),而 `cmd_mask` emission 与 routing 指令在状态的 `entry` / `during` 上集中输出(见 §4.6 emission 表 + §4.5 守卫表),不在转换 action 上单独描述。

**Priority 全局规则**(per D1 §4.2.2 仲裁优先级 + §4.4.2 失效优先级):

| Pri | 类别 | 来源 |
|:--:|---|---|
| **P0** | SAFETY override(失效请求 + kill_switch + INS critical lost)| AND-state #2 FAILSAFE_OVERLAY 主导 |
| **P1** | Pilot kill / arm_switch off | ARM_DOMAIN.disarm |
| **P2** | Pilot mode_switch 显式变化(`Pilot_Cmd_Bus.mode_switch` 边沿)| pilot intervention |
| **P3** | GCS `CMDTYPE_*`(`MODE_CHANGE` / `ARM` / `DISARM` / `TAKEOFF` / `LAND` / `RTL`)| GCS 主导 |
| **P4** | Mode-internal sequence(takeoff complete / landing touchdown / wp_reached / mission_complete) | Mode Manager 内部推进 |
| **P5** | Auto cmd_mask request(M-11 OFFBOARD 内部)| Offboard stream |

P0 借助 AND-state #2 与 ARM_DOMAIN 并行执行,转换的"override 主 mode"语义由 §4.8 实现;P1..P5 在 ARM_DOMAIN 内部按本节顺序解析。

#### 4.3.2 ARM_DOMAIN 转换枚举(Phase 2 多旋翼)

> 缩写:`PILOT.ms` = `Pilot_Cmd_Bus.mode_switch`;`PILOT.arm_sw` = `arm_switch`;`PILOT.kill_sw` = `kill_switch`;`PILOT.thr` = `throttle_stick_n01`;`GCS.ct` = `GCS_Cmd_Bus.cmd_type`;`MM.evt` = `FMS_Mission_to_ModeMgr_Bus.{wp_reached, mission_done, mission_active}`;`SF.evt` = `FMS_Safety_to_ModeMgr_Bus.{failsafe_request_active, failsafe_target_mode}`;`INS.flag` = `INS_Out_Bus.flag.bit.*`;`INS.st` = `INS_Out_Bus.INS_Status`;每个守卫见 §4.5 完整表达式。

| # | Pri | Source → Target | Trigger Event | Guard | Action(routing 指令变化已在 target 状态 entry 完成) |
|---|:--:|---|---|---|---|
| T-01 | P4 | DISARMED.INIT → DISARMED.STANDBY | `tick`(每 step 评估)| `INS.st >= INS_STATUS_READY` 持续 ≥ `ins_unready_timeout_s` 反计时(进入 ready 即可,无须 dwell)| 清 chart-internal `init_dwell_timer` |
| T-02 | P3 | DISARMED.STANDBY → ARMED.M-02 MANUAL | `GCS.ct == CMDTYPE_ARM` 或 `PILOT.arm_sw` rising edge | `arm_check_pass`(见 §4.5 G-ARM)| latch `arm_origin = PILOT|GCS`;清 lockdown |
| T-03 | P2 | DISARMED.STANDBY → ARMED.{M-02..M-12 per `PILOT.ms`} | `PILOT.arm_sw` rising edge **且** `PILOT.ms != PMODE_NONE` | `arm_check_pass` **且** entry guard of target mode(见 §4.5)| 同 T-02 |
| T-04 | P1 | ARMED.* → DISARMED.STANDBY | `PILOT.arm_sw` falling edge **或** `GCS.ct == CMDTYPE_DISARM` | `STATE != STATE_INAIR` **或** `landed_detected`(per D5 leaf)| 清 mode-internal 计时器 |
| T-05 | P1 | ARMED.M-07 LAND → DISARMED.STANDBY | `tick` | `landed_detected` **且** `t_dwell >= land_disarm_dwell_s` (FMS_PARAM.10) | 清 land profile 计时器 |
| T-06 | P2 | ARMED.M-02 MANUAL ↔ ARMED.M-03 STABILIZE | `PILOT.ms` 变到 `PMODE_STABILIZE` / `PMODE_MANUAL`(任一方向)| `mode_switch_dwell >= mode_switch_debounce_s` (FMS_PARAM.42) | 截取当前 setpoint 作 bumpless snapshot(D4 内部消费)|
| T-07 | P2 | ARMED.M-03 STABILIZE → ARMED.M-04 ALTHOLD | `PILOT.ms == PMODE_ALTHOLD` 边沿 | G-ALT(见 §4.5)+ debounce | snapshot |
| T-08 | P2 | ARMED.M-04 ALTHOLD → ARMED.M-05 POSHOLD | `PILOT.ms == PMODE_POSHOLD` 边沿 | G-POS(见 §4.5)+ debounce | snapshot |
| T-09 | P2 | ARMED.{M-04..M-12} → ARMED.M-03 STABILIZE 或 M-02 MANUAL | `PILOT.ms` 变到 `PMODE_STABILIZE` / `PMODE_MANUAL` | debounce | snapshot;表示 pilot intervention down-mode |
| T-10 | P3 | ARMED.{any non-M-06/M-07} → ARMED.M-06 TAKEOFF | `GCS.ct == CMDTYPE_TAKEOFF` **或** mission auto-takeoff request | `STATE != STATE_INAIR` **且** G-POS **且** arm 已成立 | 启动 takeoff profile(由 Mission Manager + D4)|
| T-11 | P4 | ARMED.M-06 TAKEOFF → ARMED.M-05 POSHOLD | `mission_takeoff_complete` 脉冲(由 Mission Manager 推送 via `FMS_Mission_to_ModeMgr_Bus`)| `INS.flag.altitude>=takeoff_alt_m` (FMS_PARAM.06) **且** `vel_z` 已稳定 | 清 takeoff 计时器 |
| T-12 | P3 | ARMED.{M-04..M-11} → ARMED.M-07 LAND | `GCS.ct == CMDTYPE_LAND` **或** `PILOT.ms == PMODE_LAND` | (无附加守卫;LAND 总是允许)| 启动 land profile |
| T-13 | P4 | ARMED.M-07 LAND → DISARMED.STANDBY | `landed_detected` **且** dwell ≥ `land_disarm_dwell_s` | 同 T-05 | 同 T-05 |
| T-14 | P3 | ARMED.{any non-M-08} → ARMED.M-08 RTL | `GCS.ct == CMDTYPE_RTL` **或** `PILOT.ms == PMODE_RTL` | G-POS **且** home_set | 启动 RTL 三段 sequence |
| T-15 | P4 | ARMED.M-08 RTL → ARMED.M-07 LAND | `rtl_phase == RTL_LAND_SEQ`(由 Mission Manager 推进的 RTL phase)| `near_home` | seq 推进到 land |
| T-16 | P3 | ARMED.M-05 POSHOLD ↔ ARMED.M-09 LOITER | `MM.evt.wp_reached` 在含 loiter 字段的 wp 上 → 进 LOITER;`MM.evt` 的 loiter 计时到 → 退出 LOITER | G-POS;loiter wp 字段非零 | 启停 loiter 计时器 |
| T-17 | P3 | ARMED.M-05 POSHOLD ↔ ARMED.M-10 MISSION | `PILOT.ms == PMODE_MISSION` 或 `GCS.ct == CMDTYPE_RESUME_MISSION` 进入;`MM.evt.mission_done` 退出 | G-MISSION(见 §4.5)+ debounce | 进 mission 时拉取首 wp;退出时清 mission_active |
| T-18 | P4 | ARMED.M-10 MISSION → ARMED.M-09 LOITER | `MM.evt.wp_reached` + 当前 wp 含 loiter | 同 T-16 | 同 T-16 |
| T-19 | P4 | ARMED.M-09 LOITER → ARMED.M-10 MISSION | `loiter_timer_expired` | (无)| 推进到下一 wp |
| T-20 | P4 | ARMED.M-10 MISSION → ARMED.M-08 RTL | `MM.evt.mission_complete`(若 D5 leaf 配 mission_done_action = RTL)| (无)| (auto chained RTL)|
| T-21 | P3 | ARMED.{any} → ARMED.M-11 OFFBOARD | `PILOT.ms == PMODE_OFFBOARD` 或 `GCS.ct == CMDTYPE_OFFBOARD` | G-OFFBOARD(见 §4.5)+ Auto stream `valid==1` | 进 offboard;采纳 `auto_cmd_mask_request` |
| T-22 | P3 | ARMED.{M-02 / 与 PMODE_ACRO 等价路径} → ARMED.M-12 ACRO | (条件依赖 D5 leaf 是否启用 ACRO;ACRO 仅 GCS path,per D1 §5 风险)| (D5 leaf 锁)| (D5 leaf 锁)|
| T-23 | P3 | ARMED.M-11 OFFBOARD → ARMED.M-05 POSHOLD | offboard stream stale > timeout(由 Safety Monitor 触发 TR-09)| 见 §4.8 FS-03 路径;此处 P3 是 pilot intervention,P0 是 SAFETY | (POSHOLD entry)|
| T-24 | P2 | ARMED.M-09/M-10 → ARMED.M-05 POSHOLD | `PILOT.ms == PMODE_POSHOLD` 显式 | debounce | pilot 接管 mission |

> ACRO(T-22)在 [B2 §4.4.4](../B-contracts/B2-enum-inventory.md) 中 `PilotMode` 缺 `PMODE_ACRO` 成员,因此**仅** GCS `CMDTYPE_MODE_CHANGE` 携带 `MODE_ACRO` 触发(per [D1 §5 风险](D1-fms-functional.md));若 D5 多旋翼 leaf 决定 Phase 2 不启用,T-22 行整体 disabled,M-12 状态保留为 placeholder。

#### 4.3.3 FAILSAFE_OVERLAY 转换枚举

| # | Pri | Source → Target | Trigger Event | Guard | Action |
|---|:--:|---|---|---|---|
| F-01 | P0 | NORMAL → FS-01 RTL_FAILSAFE | `SF.evt.failsafe_request_active==1 && SF.evt.failsafe_target_mode == FS01_TARGET`(对应 D1 TR-01 RC loss / TR-02 GCS loss(action=RTL)/ TR-05 LowBat / TR-07 Geofence)| (Safety Monitor 已 debounce,此处不再守卫)| latch `failsafe_state = FAIL_RC_LOSS / FAIL_GCS_LOSS / FAIL_LOW_BATTERY / FAIL_GEOFENCE`(对应触发源);写 `error_code` |
| F-02 | P0 | NORMAL → FS-02 LAND_NOW | `SF.evt` 对应 TR-06 critical battery 或 TR-11 motor fault | (Safety 已 debounce)| latch `failsafe_state = FAIL_LOW_BATTERY`(critical 子级);`error_code = ERR_BATTERY_CRITICAL` 或 `ERR_MOTOR_FAULT` |
| F-03 | P0 | NORMAL → FS-03 HOVER_HOLD | `SF.evt` 对应 TR-02 GCS loss(action=`FA_HOLD`)或 TR-09 offboard loss | (Safety 已 debounce)| `failsafe_state = FAIL_GCS_LOSS / FAIL_OFFBOARD_LOSS` |
| F-04 | P0 | NORMAL → FS-04 DISARM_FORCED | `SF.evt` 对应 TR-03 INS critical 或 TR-10 kill_switch | (immediate;无 dwell)| `failsafe_state = FAIL_INS_INVALID`(若 TR-03)/ `FAIL_NONE`(若 TR-10 kill 主动)→ `error_code = ERR_INS_INVALID` 或 `ERR_NONE` |
| F-05 | P0 | FS-01 → FS-02 | mid-RTL critical battery(TR-06 over-takes TR-05)| Safety 重判 | `failsafe_state` 升级 |
| F-06 | P0 | FS-01 → FS-04 | mid-RTL INS critical lost(TR-03)| Safety 重判 | force disarm |
| F-07 | P0 | FS-02 LAND_NOW → FS-04 | landed + 已 disarm sequence(LAND 触地后 DISARM 自动)| `landed_detected` | 推进 |
| F-08 | P5 | FS-01 / FS-03 → NORMAL | `SF.evt.failsafe_request_active==0` **且** dwell ≥ `failsafe_recovery_dwell_s` (FMS_PARAM.43) | 触发源已恢复(由 Safety 写)| 解除 overlay,resume main mode(per §4.8.2 SUSPEND/RESUME)|
| F-09 | (强制)| FS-04 → (永不退出)| (no event)| (FS-04 不可恢复 per D1 §4.4.4;只能整机 power-cycle 重新走 INIT)| — |

#### 4.3.4 ARM_DOMAIN 与 FAILSAFE_OVERLAY 的交互

- **F-01..F-04 触发时**:ARM_DOMAIN 的 main mode 状态被 SUSPEND(Stateflow `history pseudostate` 记录最后一个内部 OR-state,见 §4.8.2);ARM_DOMAIN 的 status / state / ext_state 字段 emission 被 overlay 接管;`FMS_ModeMgr_to_Shape_Bus.route_*` 改为 overlay target mode 的 routing
- **F-08 解除时**:overlay 回 NORMAL;ARM_DOMAIN 主 chart 的 history pseudostate 让 main mode 从被 SUSPEND 的位置恢复(典型:RTL 解除后回到 POSHOLD,而非 disarmed)
- **F-04 时**:overlay 不仅写 `failsafe_state` + `error_code`,还**强制** ARM_DOMAIN 整体 → DISARMED.STANDBY(via 内部隐式 ARM_DOMAIN.disarm 转换);main mode SUSPEND 状态被丢弃(不可恢复)

#### 4.3.5 Stateflow 转换冲突仲裁

同源多目标的 deterministic 顺序(per Stateflow 默认 left-to-right + Stateflow `Execute (enter) Chart At Initialization` = false):

1. **跨 AND-state**:FAILSAFE_OVERLAY 的 P0 转换在每个 chart step 中**先于** ARM_DOMAIN 评估(Stateflow 提供 explicit AND-state ordering;chart 设置 `Order=1` for FAILSAFE_OVERLAY,`Order=2` for ARM_DOMAIN)。
2. **同一 source 多目标**:Stateflow 自动按 transition 创建顺序解析;本文件 §4.3.2 表的 # 列即创建顺序(T-01 优先于 T-02,等等)。**作者实现侧**必须保证 Pri 列与表中 # 顺序一致(P0 的转换写在最前)。
3. **同时事件命中**(例如同帧 pilot ms 变化 + GCS cmd_type 变化):per D1 §4.2.2,Pilot mode_switch 优先(P2 > P3)。
4. **Pri 平级冲突**(理论上不应出现):chart 加显式 `[guard] && (event_priority_tag == X)` 区分;若发现存在(应在 H1 / H4 场景捕捉),走 §5 risk register。

### 4.4 事件 inventory

每个事件**类型**(level / edge / message / tick)与**来源 bus 字段**或**chart-internal**:

| Event 名 | 类型 | 来源 | 备注 |
|---|---|---|---|
| `pilot_ms_change` | edge(rising/falling on `Pilot_Cmd_Bus.mode_switch` enum value 改变)| `FMS_SrcSel_Bus.pilot_mode_request` | 经 D2 §4.3.1 已 latched + valid-gated |
| `pilot_arm_request` | edge | `Pilot_Cmd_Bus.arm_switch`(latched in `FMS_SrcSel_Bus`)| arm 上升沿;无 PARAM dwell(dwell 在守卫 G-ARM 中) |
| `pilot_kill` | edge | `Pilot_Cmd_Bus.kill_switch` rising | per D1 TR-10;immediate(无 debounce)|
| `gcs_cmd_arm` | message(value-trigger:`GCS_Cmd_Bus.cmd_type == CMDTYPE_ARM`)| `FMS_SrcSel_Bus.gcs_cmd_type` | per [B2 §4.4.13](../B-contracts/B2-enum-inventory.md) |
| `gcs_cmd_disarm` | message | `gcs_cmd_type == CMDTYPE_DISARM` | 同上 |
| `gcs_cmd_takeoff` | message | `gcs_cmd_type == CMDTYPE_TAKEOFF` | 同上 |
| `gcs_cmd_land` | message | `gcs_cmd_type == CMDTYPE_LAND` | 同上 |
| `gcs_cmd_rtl` | message | `gcs_cmd_type == CMDTYPE_RTL` | 同上 |
| `gcs_cmd_mode_change` | message | `gcs_cmd_type == CMDTYPE_MODE_CHANGE` + `gcs_cmd_param[0]` 携带 target `FlightMode` | per [D1 §4.2.4](D1-fms-functional.md) |
| `gcs_cmd_pause_mission` | message | `gcs_cmd_type == CMDTYPE_PAUSE_MISSION` | 给 Mission Manager;Mode Manager 不直接消费但可见(用于 chart entry guard pause-context)|
| `gcs_cmd_resume_mission` | message | `gcs_cmd_type == CMDTYPE_RESUME_MISSION` | 同上 |
| `gcs_cmd_offboard` | message | `gcs_cmd_type == CMDTYPE_OFFBOARD` | 同上 |
| `safety_failsafe_request` | level | `FMS_Safety_to_ModeMgr_Bus.failsafe_request_active` | P0 触发;target mode 在 `failsafe_target_mode` 字段 |
| `mission_wp_reached` | edge(pulse from Mission Manager)| `FMS_Mission_to_ModeMgr_Bus.wp_reached` | per [D2 §4.2.1](D2-fms-structural.md) |
| `mission_complete` | edge | `FMS_Mission_to_ModeMgr_Bus.mission_done` | 同上 |
| `mission_takeoff_complete` | edge | `FMS_Mission_to_ModeMgr_Bus.takeoff_complete`(若 B1 锁定 / 或 internal between Mission ↔ Mode Mgr;D2 §4.2.1 字段类别 (c) 内部 phase)| 由 Mission Manager 推送 |
| `landed_detected` | edge | derived from Plant `Plant_States_Bus.landed_flag`(per A3 §4.4.4 / C1 §4.x;D5 leaf 决定如何计算)→ via `FMS_SrcSel_Bus` echo / 或直消费 INS-derived altitude+vel | per D1 TR-13(隐式);Phase 2 多旋翼用 D5 leaf 实现 |
| `ins_attitude_valid_lost` | edge(falling on `INS_Flag.bit.attitude_valid`)| `INS_Out_Bus.flag` | 紧急触发;由 Safety Monitor 转 P0(本 chart 不直消费此 edge,改 走 `safety_failsafe_request`) |
| `ins_position_valid_lost` | edge | 同上 | 同 attitude;由 Safety 升 P0 |
| `ins_status_change` | edge | `INS_Status` 跨 `INS_STATUS_READY` 边界 | 给 INIT → STANDBY 转换(T-01) |
| `tick` | implicit(每 chart step 的 `during`)| chart-internal | for dwell / debounce / sequence 推进 |
| `mode_switch_debounce_done` | timer | chart-internal `t_ms_debounce >= mode_switch_debounce_s` (FMS_PARAM.42)| pilot mode_switch 转换守卫的部件 |
| `failsafe_recovery_dwell_done` | timer | chart-internal `t_fs_recover >= failsafe_recovery_dwell_s` (FMS_PARAM.43) | overlay 解除转换守卫的部件 |
| `init_complete` | edge | derived from `INS_Status >= INS_STATUS_READY` | T-01 触发器(化简为 ins_status_change 的子集)|
| `auto_cmd_mask_request_change` | message | `FMS_SrcSel_Bus.auto_cmd_mask_request` | M-11 OFFBOARD entry / during 消费 |

事件**来源边界**统一为 `FMS_SrcSel_Bus`(per D2 §4.2.1)+ `INS_Out_Bus`(per D2 §4.3.2 直消费)+ `FMS_Safety_to_ModeMgr_Bus` + `FMS_Mission_to_ModeMgr_Bus`;chart 不直接读 `Pilot_Cmd_Bus` / `GCS_Cmd_Bus` 等外部 bus(走 Source Selector,per D2 §4.3.1)。例外:`Pilot_Cmd_Bus.kill_switch` 可由 Safety Monitor 直传 → 转为 `safety_failsafe_request` 的 P0 路径,与 D1 §4.2.2 第 3 行优先级一致。

### 4.5 守卫(Guards)

守卫表达式以**声明式**给出,引用 [B3 FMS_PARAM](../B-contracts/B3-parameter-schema.md) 字段名(不锁数值)+ B2 enum 成员名 + B1 字段名。`&&` / `||` / `!` = 逻辑与/或/非;`==` 是 enum 等值;`>=` / `>` / `<=` / `<` 是数值比较。

| 守卫名 | 适用转换 | 表达式 | 来源 |
|---|---|---|---|
| **G-ARM** | T-02 / T-03(arm 进入)| `(PILOT.thr <= arm_throttle_thr_n01) && (PILOT.kill_sw == 0) && (INS.st >= INS_STATUS_READY) && (INS.flag.attitude_valid == 1) && (arm_safety_check_mask 全位通过)` | D1 §4.4 TR-08 + B3 FMS_PARAM.02/.05 + A6 §4.5.3 INS gate + per D5 leaf 增补 mass / battery 检查 |
| **G-ALT** | T-07 + 其他进入 ALTHOLD 的转换 | `(INS.st >= INS_STATUS_READY) && (INS.flag.attitude_valid == 1) && (INS.flag.heading_valid == 1)`(z-vel 由 baro/ vertical INS 派生;D6 §4.2.2 锁 BIT_VEL 仅水平 vel 需 velocity_valid;z-vel 守卫 D5 leaf 决定)| D6 §4.2.2 + §4.5 |
| **G-POS** | T-08 / T-10 / T-14 / T-16 / T-17 / 其他 POS 类进入 | `G-ALT && (INS.flag.position_valid == 1) && (INS.flag.velocity_valid == 1)` | D6 §4.5 INS 政策 + A6 §4.5.3 |
| **G-MISSION** | T-17 进入 MISSION | `G-POS && (mission_state ∈ {MISSION_LOADED, MISSION_PAUSED}) && (wp_count >= 1)` | D1 §4.5.1 + B2 §4.4.8 |
| **G-OFFBOARD** | T-21 进入 OFFBOARD | `G-POS && (Auto_Cmd_Bus.valid == 1) && (Auto stream 不 stale per gcs_loss_timeout_s 复用)` | D1 §4.4 TR-09 + §4.2.4 表行 M-11 |
| **G-DEBOUNCE-MS** | T-06..T-09 / T-17 / T-24 等 pilot mode_switch 触发的转换的**附加** | `t_ms_debounce >= mode_switch_debounce_s` (FMS_PARAM.42)| D1 §4.4.3 debounce 类 + B3 FMS_PARAM.42 |
| **G-FS-RECOVER** | F-08 overlay 解除 | `failsafe_request_active == 0` 持续 `>= failsafe_recovery_dwell_s` (FMS_PARAM.43)| D1 §4.4.3 失效解除回滞 + B3 FMS_PARAM.43 |
| **G-INS-READY** | T-01 离开 INIT | `INS.st >= INS_STATUS_READY`(进入 ready 即可,不需要 dwell)| A6 §4.5.3 + B2 §4.4.10 |
| **G-LANDED** | T-04 / T-05 / T-13 / F-07 disarm path | `landed_detected == 1`(由 Plant landing detector / D5 leaf 决定数值条件)| D5 leaf 锁;D3 引用谓词名 |
| **G-NOT-INAIR** | T-04 disarm | `STATE != STATE_INAIR` ⇔ `landed_detected == 1` 或 `STATE == STATE_GROUND` | D1 §4.1.1 + B2 §4.4.2 |
| **G-NEAR-HOME** | T-15 RTL → LAND | `dist_to_home_m <= rtl_acceptance_radius_m` (D5 leaf 锁,典型复用 `wp_acceptance_radius_m` FMS_PARAM.26)| D5 leaf;D1 §4.3.4 RTL profile |
| **G-HOME-SET** | T-14 进入 RTL | `home_set == 1`(由 GCS `CMDTYPE_SET_HOME` 设置或由 PARAM .30..32 default 加载)| D1 §4.6.3 + B3 FMS_PARAM.30..32 |

**Bumpless entry guard**(per D1 §4.3.3 + D5 leaf):每个状态的 entry action 中保存当前 setpoint(`pos_cmd` / `vel_cmd` / `att_cmd` 等)的"前一帧值"作 snapshot,经 `FMS_ModeMgr_to_Shape_Bus.bumpless_snapshot_*` 透传给 D4(D4 据此实现 setpoint 平滑过渡,典型 ramp 一帧或多帧);D3 在 chart action 中只**保存**,**不**生成新 setpoint。

### 4.6 cmd_mask emission per state(D3 ↔ D6 closure)

D3 在每个 ARMED 状态的 `entry` 上一次性写 `cmd_mask` + `route_*` 路由指令,经 `FMS_ModeMgr_to_Shape_Bus`(D2 §4.2.1)给 D4。Bit 数值(0..7)与 工作位号 per [B2 §4.4.14](../B-contracts/B2-enum-inventory.md);**位语义**(置位时 Controller 行为)per [D6 §4.2](D6-fms-controller-interface.md);**Truth-table 合法性** per D6 §4.4.1;**INS-invalid 政策** per D6 §4.5。本表满足 [D6 §4.6.1](D6-fms-controller-interface.md) 列头与契约形状。

| Mode (state) | POS (bit 0) | VEL (bit 1) | ACC (bit 2) | ATT (bit 3) | RATE (bit 4) | YAW (bit 5) | YAW_RATE (bit 6) | THR (bit 7) | 8-tuple | 对应 D6 §4.4.1 行 / 备注 |
|---|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|---|---|
| **DISARMED.INIT / STANDBY** | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | `(0,0,0,0,0,0,0,0)` | reset / disarmed safe state(per A6 §4.4.2 + D6 MX-1 / D6 §4.7) |
| **ARMED.M-02 MANUAL** | 0 | 0 | 0 | 0 | 1 | 0 | 1 | 1 | `(0,0,0,0,1,0,1,1)` | RATE + YAW_RATE + THR direct(D6 §4.4.1 行 2) |
| **ARMED.M-03 STABILIZE** | 0 | 0 | 0 | 1 | 0 | 0 | 1 | 1 | `(0,0,0,1,0,0,1,1)` | ATT + YAW_RATE + THR direct(D6 §4.4.1 行 3) |
| **ARMED.M-04 ALTHOLD** | 0 | 1 | 0 | 1 | 0 | 0 | 1 | 0 | `(0,1,0,1,0,0,1,0)` | VEL(z) + ATT(roll/pitch) + YAW_RATE;throttle 闭环(D6 §4.4.1 行 4)|
| **ARMED.M-05 POSHOLD** | 1 | 1 | 1 | 0 | 0 | 1 | 1 | 0 | `(1,1,1,0,0,1,1,0)` | POS + VEL + ACC + YAW + YAWR;ATT 由 thrust-vector cascade 隐式推导(per D6 §4.3.3 / §4.3.6;BIT_ATT=0 表示 FMS 不显式提供姿态参考)— 与 D4 §4.2.1 行 M-05 一致 |
| **ARMED.M-06 TAKEOFF** | 1 | 1 | 0 | 0 | 0 | 1 | 0 | 0 | `(1,1,0,0,0,1,0,0)` | POS + VEL + YAW;垂直 climb 走 pos+vel z-component(profile 提供);ATT 由 thrust-vector cascade 隐式推导 — 与 D4 §4.2.1 行 M-06 一致 |
| **ARMED.M-07 LAND** | 1 | 1 | 0 | 0 | 0 | 1 | 0 | 0 | `(1,1,0,0,0,1,0,0)` | descent profile = pos hold(水平)+ vel(z) + YAW;ATT 由 thrust-vector cascade 隐式推导(hover descent)— 与 D4 §4.2.1 行 M-07 一致 |
| **ARMED.M-08 RTL** | 1 | 1 | 1 | 0 | 0 | 1 | 0 | 0 | `(1,1,1,0,0,1,0,0)` | POS + VEL + ACC + YAW(face-home);ATT 由 thrust-vector cascade 隐式推导;final descent 转 M-07(LAND)— 与 D4 §4.2.1 行 M-08 一致 |
| **ARMED.M-09 LOITER** | 1 | 0 | 0 | 0 | 0 | 1 | 0 | 0 | `(1,0,0,0,0,1,0,0)` | POS + YAW(wp 处停留);ATT 由 thrust-vector cascade 隐式推导 — 与 D4 §4.2.1 行 M-09 一致 |
| **ARMED.M-10 MISSION** | 1 | 1 | 1 | 0 | 0 | 1 | 0 | 0 | `(1,1,1,0,0,1,0,0)` | POS + VEL + ACC + YAW(三阶 FF);ATT 由 thrust-vector cascade 隐式推导 — 与 D4 §4.2.1 行 M-10 一致 |
| **ARMED.M-11 OFFBOARD** | per Auto | per Auto | per Auto | per Auto | per Auto | per Auto | per Auto | per Auto | per `auto_cmd_mask_request` 经 D6 §4.4 互斥过滤 | D6 §4.4.1 行 9..14 中合法子集;D3 转换 T-21 entry action 拷贝 + 过滤 |
| **ARMED.M-12 ACRO** | 0 | 0 | 0 | 0 | 1 | 0 | 1 | 1 | `(0,0,0,0,1,0,1,1)` | 同 M-02 MANUAL;ACRO 是 rate-stick 直传 |
| **FAILSAFE_OVERLAY.NORMAL** | (透明 — 不写)| | | | | | | | (透明)| overlay 默认 — main mode 写 |
| **FAILSAFE_OVERLAY.FS-01 RTL_FAILSAFE** | 1 | 1 | 1 | 0 | 0 | 1 | 0 | 0 | `(1,1,1,0,0,1,0,0)` | overlay 接管 = M-08 RTL schema(home as wp);ATT 由 thrust-vector cascade 隐式推导 — 与 D4 §4.2.2 行 FS-01 一致 |
| **FAILSAFE_OVERLAY.FS-02 LAND_NOW** | 1 | 1 | 0 | 0 | 0 | 1 | 0 | 0 | `(1,1,0,0,0,1,0,0)` | overlay 接管 = M-07 LAND schema(controlled descent);ATT 由 thrust-vector cascade 隐式推导(hover descent)— 与 D4 §4.2.2 行 FS-02 一致 |
| **FAILSAFE_OVERLAY.FS-03 HOVER_HOLD** | 1 | 1 | 1 | 0 | 0 | 1 | 1 | 0 | `(1,1,1,0,0,1,1,0)` | overlay 接管 = POSHOLD at current pos;ATT 由 thrust-vector cascade 隐式推导 — 与 D4 §4.2.2 行 FS-03 一致 |
| **FAILSAFE_OVERLAY.FS-04 DISARM_FORCED** | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | `(0,0,0,0,0,0,0,0)` | all-zero(per D6 §4.7 强制 disarmed-equivalent) |

**契约形状满足**(per D6 §4.6.1):
- 每行 emission **必须** ∈ D6 §4.4.1 truth-table 的"合法"集合 → 上表全部行已交叉对照 D6 §4.4.1 的"合法 +" 行(`(0,0,0,0,0,0,0,0)`、`(0,0,0,0,1,0,1,1)`、`(0,0,0,1,0,0,1,1)`、`(0,1,0,1,0,0,1,0)`、`(1,0,0,0,0,1,0,0)`、`(1,1,0,0,0,1,0,0)`、`(1,1,1,0,0,1,0,0)`、`(1,1,1,0,0,1,1,0)`)— POS-driven modes(M-05/M-06/M-08/M-09/M-10/FS-01/FS-03)的 BIT_ATT=0 走 D6 §4.3.3 隐式 cascade 路径(thrust-vector→ATT(roll/pitch));LAND-family(M-07/FS-02)同样走隐式 cascade(hover descent)。
- D3 不 emit `非法 MX-*` 组合(D6 §4.4)。
- 单一写者:Stateflow `entry` 上写,`during` 上保持(`during` 不重写,除 OFFBOARD M-11 — Auto_Cmd_Bus 变化时由 `auto_cmd_mask_request_change` event 触发 entry action 重新过滤一次)。Output Assembler 不修改(per D2 §4.6.2)。
- `Auto_Cmd_Bus.cmd_mask_request` 在 M-11 OFFBOARD 内的采纳:T-21 entry action 中**先**经 D6 §4.4 互斥过滤 + §4.5 INS validity 政策过滤;**通过的位**进入 `cmd_mask`;**不通过的位**丢弃,并写 `error_code = ERR_FMS_CMD_INCONSISTENT` (per D6 §4.4 MX-9 类比;具体 ErrorCode 名由 B2 / D6 / E1 锁,本文件留 B2 §4.4.9 现有 `ERR_INTERNAL` 作 fallback)。
- Reset 时 = 全 0(per A6 §4.4.2 + D1 §4.1.3 + D6 §4.7)。

**Setpoint slots used**(对应 D6 §4.6.2 D4 hand-off 表 — D4 fills numerics):每个置位 bit 对应一个 setpoint 字段(per D6 §4.6.2),D4 据 `FMS_ModeMgr_to_Shape_Bus.route_*` 路由指令决定字段值来源(如 M-05 POSHOLD 的 `pos_cmd_ned_m` 来自 INS 当前 pos 锁存,M-09 LOITER 的来自 wp 派生;来源类别 per D1 §4.2.4 表)。

### 4.7 Reset / Init 行为(per A6)

#### 4.7.1 Chart entry state

per [A6 §4.4.2](../A-architecture/A6-init-reset-contract.md) + [D1 §4.1.3](D1-fms-functional.md) + [D2 §4.8.1](D2-fms-structural.md):

- chart 的 default initial transition 指向 `ARM_DOMAIN.DISARMED.INIT`
- `FAILSAFE_OVERLAY` 的 default initial transition 指向 `NORMAL`
- 二者由 chart 的 `Default Transition` 与 AND-state initial sub-state 标记设置(per Stateflow 默认机制)
- chart 内部所有 timer / counter / last-frame snapshot 变量在 `FMS_init` 上清零(per [D2 §4.8.1](D2-fms-structural.md) Mode Manager 行)

#### 4.7.2 Reset 后 emission 稳态

per [A6 §4.4.2](../A-architecture/A6-init-reset-contract.md) `FMS_Out_Bus` reset 类别表(行被本表覆盖到);Output Assembler 在 reset 上写下表稳态,这些值与 chart entry state(DISARMED.INIT)的 §4.2 映射**一致**:

| `FMS_Out_Bus` 字段 | reset 稳态 | A6 类别 | chart 中对应 leaf 状态写值 |
|---|---|---|---|
| `mode` | `MODE_NONE` | HARDCODED | DISARMED.INIT / STANDBY |
| `status` | `STATUS_DISARM` | HARDCODED | 同上 |
| `state` | `STATE_DISARM` | HARDCODED | 同上 |
| `ext_state` | `EXT_STATE_LANDED` | HARDCODED | 同上 |
| `ctrl_mode` | `CMODE_DISARMED` | HARDCODED | 同上 |
| `cmd_mask` | 全 0 | HARDCODED zero (safe state)| §4.6 INIT/STANDBY 行 |
| `failsafe_state` | `FAIL_NONE` | HARDCODED(若字段存在 per B1)| FAILSAFE_OVERLAY.NORMAL |
| `error_code` | `ERR_NONE` | HARDCODED | 默认无错 |

#### 4.7.3 Local reset(运行期)

per [A6 §4.5.2](../A-architecture/A6-init-reset-contract.md) global vs local reset:

- **F-04 FS-04 触发** → chart 对 ARM_DOMAIN 发 implicit "force-disarm" transition(等同 T-04 但跳过 G-NOT-INAIR);main mode SUSPEND state 丢弃(history pseudostate 清);进入 DISARMED.STANDBY(若 INS 恢复)/ DISARMED.INIT(若 INS critical lost)
- **重新 arm 后**:走 T-02 / T-03,清 chart-internal 上次失效计时器
- chart 的 `reset` 输出位(写到 `FMS_Out_Bus.reset` 若 B1 锁定字段存在;per D2 §4.6.4)在 F-04 entry 上 pulse 一次,通知其他 5 个子模块同步走 local reset

#### 4.7.4 INS-validity-lost 期间的 emission

per [D6 §4.5](D6-fms-controller-interface.md) "FMS 应有的良民行为" — chart 的 SAFETY override 路径(F-03 / F-04)处理 INS lost,因此从 `FMS_Out_Bus` 边界看:
- INS attitude lost(critical) → F-04 FS-04 → cmd_mask = 全 0(D6 §4.5 表行 "INS 完全失效" / "INS 姿态失效")
- INS position lost(非 critical)→ Safety Monitor 评估当前 mode 是否依赖 POS;若依赖 → F-01 FS-01(降级到 RTL 用 dead-reckoning)/ F-03 FS-03(若 D5 leaf 配 hover hold);若不依赖(M-02 / M-03 / M-04 / M-12)→ NORMAL,main mode 继续

### 4.8 Failsafe overlay 结构

#### 4.8.1 AND-state vs OR-state 抉择

| 选项 | 描述 | 结果 |
|---|---|---|
| **(A) OR-state**(失效与 main mode 互斥)| FS-01..FS-04 与 M-02..M-12 同层 OR-state;失效触发即 main → FS-X 转换,失效解除 → FS-X → 某个 main 转换 | **拒绝**:解除后 main mode 内部 dwell / phase 状态丢失;若 RTL 解除回 POSHOLD 后,POSHOLD 不知道之前是从 MISSION 进的还是从 STABILIZE 进的;丢失 mission resume 能力 |
| **(B) AND-state overlay**(失效与 main mode 并行)| FAILSAFE_OVERLAY 是 chart 的第二个 AND-state region,与 ARM_DOMAIN 并行;NORMAL 默认透明,FS-X 接管 emission;main mode 在 ARM_DOMAIN 中**保持**(SUSPEND in concept,Stateflow 中靠 history pseudostate)| **采用**:解除后 main mode 自然 resume;chart 拓扑清晰(Mode Manager 真值在两个正交 region 上)|

D3 选 **(B) AND-state overlay**。

#### 4.8.2 SUSPEND / RESUME 实现(Stateflow history pseudostate)

per Stateflow `history` 机制:

- `ARMED` super-state 配置 `History` pseudostate 在其中;NORMAL → FS-X 触发时,ARM_DOMAIN 仍处 ARMED super-state(因为 FS-X 不在 ARM_DOMAIN 中,FAILSAFE_OVERLAY 与 ARM_DOMAIN 并行,FS-X 不引起 ARM_DOMAIN 内部转换)
- **关键不变量**:F-01..F-04 触发**不**导致 ARM_DOMAIN 中的 main mode OR-state 退出;只是 emission 被 overlay 覆盖。F-08 解除 → overlay 回 NORMAL → main mode 的 emission 自然恢复
- **例外**:F-04 FS-04 同时触发隐式 ARM_DOMAIN.disarm transition(如 §4.7.3),此时 main mode 真的退出

#### 4.8.3 Overlay 的 emission 接管协议

| 字段 | NORMAL 写者 | FS-* 写者(when overlay active)|
|---|---|---|
| `mode` | ARM_DOMAIN main mode | overlay FS-X(per §4.2 表)|
| `status` / `state` / `ext_state` | ARM_DOMAIN | overlay |
| `ctrl_mode` | ARM_DOMAIN | overlay |
| `failsafe_state` | NORMAL writes `FAIL_NONE` | overlay writes per §4.2 |
| `error_code` | (不写,保持上一帧或清除规则,见 §5 risk D1)| overlay writes |
| `cmd_mask` + `route_*` | ARM_DOMAIN main mode | overlay(per §4.6 FS-* 行)|

实现侧:每个 chart 字段的写入由"最后写者"决定;Stateflow 默认机制下 AND-state 按 `Order` 顺序执行(NORMAL Order=1, FS-* Order=2 within FAILSAFE_OVERLAY;ARM_DOMAIN Order=2 全局),overlay 写在 ARM_DOMAIN 之后,自然实现"接管"语义。

#### 4.8.4 Overlay 与 SAFETY 优先级(D1 §4.2.2 P1)

D1 §4.2.2 第 1 行 "SAFETY override" → 本 chart 的 P0(F-01..F-04);本 chart 不再实现一个独立的"override 仲裁器",而是依赖:
- AND-state ordering(FAILSAFE_OVERLAY 先于 ARM_DOMAIN)
- emission 接管协议(§4.8.3)
- Pri 顺序(§4.3.1 P0 > P1..P5)

三者合一确保任意 main mode 期间 SAFETY 触发都能立即接管 emission。

### 4.9 Hand-off 给 D4 / D6 / E1

D3 此次 batch(Wave 8)co-seal 给三位 sibling 的具体 hand-off 内容:

#### 4.9.1 → D4

- **bus**:`FMS_ModeMgr_to_Shape_Bus`(per D2 §4.2.1 字段类别)
  - `active_mode`:本表 §4.2 中 chart leaf 状态对应的 `FlightMode` 成员名(D4 据此选 routing 树分支)
  - `ctrl_mode_hint`:本表 §4.2 中对应的 `CtrlMode` 成员名
  - `route_*`:每个 setpoint 字段的来源类别(per D1 §4.2.4 表 — `HOLD` / `MISSION` / `PILOT_OFFSET` / `AUTO_TARGET`),由本 chart 状态 `entry` 写;状态对照见 §4.6 + §4.2
  - `enable_*`:每个整形原语的开关(`enable_jerk_lim` / `enable_tilt_lim` / `enable_takeoff_profile` / `enable_landing_profile` / `enable_rtl_profile`)
  - `bumpless_snapshot_*`:per §4.5 bumpless guard 段,转换 entry 上保存的当前 setpoint 快照(可选,D5 leaf 决定具体使用)
- **D4 据此实现**:整形公式(D4 §4.x)+ takeoff/landing/RTL profile 数学(D4 §4.x)+ cmd_mask 与字段值的同步发布(D4 §4.x,per D6 §4.6.2)
- D4 **不**重新决定 `cmd_mask` 位组合(D3 在 §4.6 已锁);D4 仅根据 D3 的 `cmd_mask` + `route_*` 填字段值

#### 4.9.2 → D6

D3 §4.6 emission 表满足 [D6 §4.6.1](D6-fms-controller-interface.md) D3 hand-off 表的列头与契约形状:
- 列头 = 8 个 `MASK_BIT_*`(per D6 §4.6.1)
- 每行 ∈ D6 §4.4.1 合法 truth-table
- M-11 OFFBOARD 行的 `auto_cmd_mask_request` 采纳走 D6 §4.4 互斥过滤(per §4.6 注)
- reset = 全 0(per D6 §4.7)
- single-writer(D2 §4.6.2)

D6 据此核对 §4.6.1 表的"D3 fills"列,在 D6 评审中确认完整性。

#### 4.9.3 → E1

D3 emit 的 cmd_mask + ctrl_mode 经 Output Assembler → `FMS_Out_Bus`(D2 §4.6) → Controller 边界(per A4 RB-05 latch)→ E1 Setpoint Decode(per [E2 §4.1.1](../E-controller/E2-controller-structural.md));E1 §4.2 trim rules 据此裁剪环路。D3 反向**不**约束 E1 算法(loop 内 trim / fallback / windup 全在 E1 范围)。

#### 4.9.4 → D5(Wave 9)

D3 chart 拓扑 + 转换 + 守卫**全部 shared**(macro);D5 leaf 注入仅通过 PARAM(per A5 + D2 §4.7):
- `arm_safety_check_mask`(FMS_PARAM.05)位语义由 D5 leaf 锁(多旋翼专属 arm 检查项)
- `tilt_lim_rad`(FMS_PARAM.38)/ `vel_xy_lim_mps`(.33)/ `vel_z_*_lim_mps`(.34/.35) 等限值数值由 D5 leaf 锁
- `landed_detected` 谓词的具体计算公式(高度 + 垂速 + throttle 阈值)由 D5 leaf 锁
- `mission_done_action`(若 chained RTL,见 T-20)由 D5 leaf 锁
- M-12 ACRO 是否启用(per [D1 §5 风险](D1-fms-functional.md))由 D5 leaf 决定

## 5. 已知风险与悬而未决问题

- **`error_code` 清除时机未明确**(continuation of D1 §5 同名条目)
  - 影响:§4.2 表 / §4.6 注释中,error_code 在触发态 entry 上写非 NONE 值;但 transitions T-04 / T-13 disarm 时是否清回 ERR_NONE 未定。若不清,disarm 后的 STANDBY 状态会带着上次错误码,污染 GCS 显示
  - 处置:推荐方案 = 在 ARM_DOMAIN 进入 DISARMED.STANDBY 的所有 transitions(T-04 / T-13 / F-04→DISARM)的 entry action 上清 `error_code = ERR_NONE`;F-04 不可恢复路径除外(让 error_code 持续到下次 power-cycle 显示告警来源)。具体由 D5 leaf + GCS UI 协同决定;暂列 open
- **M-06 TAKEOFF 与 M-07 LAND 的 `FlightMode` enum 数值复用 `MODE_NONE` 与 `MODE_LAND`**
  - 影响:D1 §4.1.1 标 M-06 的 `FlightMode` = `MODE_NONE` 派生 + `ext_state=TAKING_OFF`;D3 §4.2 表与之一致。但若 firmware 后续 mirror 一个独立 `MODE_TAKEOFF` enum 成员(per [B2 §4.4.5 末成员追加规则](../B-contracts/B2-enum-inventory.md) P-2),本表 M-06 行需同步;Phase 2 不阻塞
  - 处置:延后到 firmware mirror 后(B5 触发);D3 §10 changelog 同步
- **`landed_detected` 谓词归属(D5 vs C 区 Plant)**
  - 影响:§4.5 G-LANDED 谓词由"D5 leaf 锁数值条件";但具体计算来源是 INS-derived altitude+vel(由 INS_Out_Bus 派生)还是 Plant_States_Bus 直接给出的 `landed_flag`(D2 §4.3.1 Source Selector 路径)未明
  - 处置:延后到 D5 + C1 leaf detail(C3 多旋翼 Plant leaf,Wave 9);D3 仅 declare 谓词名 `landed_detected` 与 G-LANDED 守卫名,数值计算不锁
- **history pseudostate 在 F-04 时清空策略**
  - 影响:§4.8.2 锁 F-04 触发时清 history(main mode 不可 resume);Stateflow `history` 默认行为是"AND-state 退出时清"。F-04 路径中 ARM_DOMAIN 走隐式 disarm transition,等同 ARM_DOMAIN 退出 ARMED super-state,history 自然清。但若 Stateflow 实现侧用 `Deep History` 而 main mode 是嵌套 OR-state,清的范围需显式声明
  - 处置:推荐 `Shallow History` 在 ARMED super-state(只 resume 顶层 mode,不 resume mode 内部子状态);M-08 RTL / M-10 MISSION 内部的 phase 不靠 history 而靠 Mission Manager 持有(D2 §4.3.4)。具体由 D5 leaf 实现侧锁
- **M-11 OFFBOARD 的 `cmd_mask` 行运行时 evaluation 与 entry-once 之间的折中**
  - 影响:§4.6 注说明 OFFBOARD 在 entry 一次过滤 + `auto_cmd_mask_request_change` 事件触发重新过滤;但若 OFFBOARD 期间 INS validity 单独变化(进 §4.7.4 INS-position lost 时),应触发 chart 重判但 cmd_mask 是否需要在 OFFBOARD `during` 上动态调整未明确(若不调,Controller 见到非法 mask 走 D6 §4.4 fallback)
  - 处置:推荐 = OFFBOARD `during` 也评估 INS validity 变化;若变化,触发 internal re-filter event。详细机制由 D5 leaf 决定;Phase 2 闭环切片若不含 OFFBOARD INS 退化场景,Phase 2 暂用 entry-once
- **G-DEBOUNCE-MS 在多个 P2 转换上重复使用,可能在快速 mode_switch 序列中造成转换抑制**
  - 影响:`mode_switch_debounce_s` 默认 `0.05 s` (B3 FMS_PARAM.42),约 2.5 个 chart step;若 pilot 快速摇 mode_switch,G-DEBOUNCE-MS 可能阻挡某些合法转换。Phase 2 默认 0.05 s 对人类手指足够,但若 GCS 自动序列触发 mode_switch 则需 D5 leaf 调
  - 处置:open;由 D5 leaf 决定数值;H1 / H4 加用例验证
- **OFFBOARD M-11 的 `ctrl_mode` 推导规则**
  - 影响:§4.2 表 M-11 行 ctrl_mode 标 "per `auto_cmd_mask_request`";但 D6 §4.6.1 D3 hand-off 列头不要求 `ctrl_mode` 列。具体推导规则(从置位的 mask 位推 CtrlMode 成员)未在本文锁
  - 处置:open;由 D6 在 §4.6.1 备注或 §4.6.3 E1 hand-off 中锁定推导规则(典型:`POS=1 → CMODE_AUTO_POS`;`VEL=1 && POS=0 → CMODE_AUTO_VEL`;`ATT=1 && VEL=0 && POS=0 → CMODE_MANUAL_ANGLE`;`RATE=1 && ATT=0 → CMODE_MANUAL_RATE`;`THR=1 && all-others=0 → CMODE_THROTTLE_PASSTHROUGH`)
- **TR-04 GPS lost 在 chart 中的对应 transition 未在 §4.3 显式列出**
  - 影响:D1 §4.4.1 TR-04 是"在 M-05/M-08/M-09/M-10 期间 → M-04 ALTHOLD"的部分降级;Safety Monitor 推 `failsafe_request_active` + `failsafe_target_mode = MODE_ALTHOLD`?但 D3 §4.3 中 FS-* 没有 ALTHOLD 行(FS-01..FS-04 是 RTL/LAND/HOVER/DISARM,不含降级到 ALTHOLD)
  - 处置:推荐 = TR-04 不走 FAILSAFE_OVERLAY,改为 Safety Monitor 直接以 P3 优先级触发"GCS-style mode change to MODE_ALTHOLD"(类似 T-09 的 down-mode),不进 overlay。D3 §4.3 在该路径上加一条 T-25(Phase 2 多旋翼如启用 GPS-only 降级);具体走法 D5 leaf 锁
- **chart 内 `tick` event 的 fired-rate vs Stateflow `during` semantics**
  - 影响:§4.4 标 `tick` = "每 chart step 的 `during`",这是常规 Stateflow 行为;但若实现侧把 tick 显式建模为定时器输出 event 触发,可能引入隐式延迟一个 chart step。Phase 2 不阻塞,但实现层需对齐
  - 处置:实现侧约定;不在设计层锁;在 D5 leaf / 实现阶段抓取

## 6. 退出条件复核

对照 [`00-design-plan.md §4.D`](../00-design-plan.md) D3 退出条件:

> **D3 退出条件原文**:Stateflow 状态/转换/事件/守卫定稿;状态与 `VehicleStatus`/`VehicleState`/`ctrl_mode` 映射表

| # | 退出条件原文(分项)| 本文档依据 | 状态 |
|---|---|---|---|
| 1 | Stateflow 状态定稿 | §4.1.1 chart 顶层结构(2 个 AND-state region;ARM_DOMAIN 含 DISARMED super(INIT + STANDBY)+ ARMED super(M-02..M-12 全 12);FAILSAFE_OVERLAY 含 NORMAL + FS-01..FS-04;§4.1.3 共 22 leaf states;§4.1.2 拓扑决策 | 满足 |
| 2 | Stateflow 转换定稿 | §4.3.2 ARM_DOMAIN 24 条转换 T-01..T-24 + §4.3.3 FAILSAFE_OVERLAY 9 条 F-01..F-09 + §4.3.4 跨 region 交互 + §4.3.5 冲突仲裁 | 满足 |
| 3 | Stateflow 事件定稿 | §4.4 事件 inventory(22 个 event 类型 + 来源 bus 字段 + 类型 level/edge/message/tick/timer | 满足 |
| 4 | Stateflow 守卫定稿 | §4.5 守卫表(12 个守卫 G-ARM / G-ALT / G-POS / G-MISSION / G-OFFBOARD / G-DEBOUNCE-MS / G-FS-RECOVER / G-INS-READY / G-LANDED / G-NOT-INAIR / G-NEAR-HOME / G-HOME-SET + bumpless guard 段)— declarative,引用 B3 PARAM 名,不锁数值 | 满足 |
| 5 | 状态 ↔ `VehicleStatus` 映射 | §4.2 表 STATUS 列(覆盖 `STATUS_DISARM` / `STATUS_STANDBY` / `STATUS_ARM` 全 3 成员)| 满足 |
| 6 | 状态 ↔ `VehicleState` 映射 | §4.2 表 STATE 列(覆盖 `STATE_DISARM` / `STATE_STANDBY` / `STATE_LOCKDOWN` / `STATE_INAIR` / `STATE_GROUND` 全 5 成员;LOCKDOWN 由 FS-04 行覆盖)| 满足 |
| 7 | 状态 ↔ `ctrl_mode` 映射 | §4.2 表 `CtrlMode` 列(覆盖 `CMODE_DISARMED` / `CMODE_MANUAL_ANGLE` / `CMODE_MANUAL_RATE` / `CMODE_AUTO_VEL` / `CMODE_AUTO_POS` / `CMODE_THROTTLE_PASSTHROUGH` 全 6 成员;THROTTLE_PASSTHROUGH 由 M-02 manual sub-variant / M-12 ACRO 覆盖)| 满足 |

**额外覆盖**(超出退出条件原文,但 brief 要求):
- 状态 ↔ `VehicleExtState` 映射 — §4.2 表 EXT_STATE 列(覆盖全 6 成员)
- 状态 ↔ `FlightMode` 映射 — §4.2 表 `FlightMode` 列(覆盖 10 成员中 9 项;`MODE_NONE` 同时承载 IDLE + TAKEOFF;`MODE_POSHOLD` 同时承载 M-05 + M-09;`MODE_ACRO` 由 M-12 覆盖,M-12 是否启用 D5 锁)
- 状态 ↔ `FailsafeState` 映射 — §4.2 表 FailsafeState 列(覆盖 `FAIL_NONE` / `FAIL_RC_LOSS` / `FAIL_GCS_LOSS` / `FAIL_LOW_BATTERY` / `FAIL_GEOFENCE` / `FAIL_INS_INVALID` / `FAIL_OFFBOARD_LOSS` 全 7 成员)
- cmd_mask emission per state — §4.6 表(满足 D6 §4.6.1 列头 + §4.4.1 truth-table 合法集合)

## 7. 下游影响

按 [`01-design-relationships.md §4.4`](../01-design-relationships.md) D3 出边 + Wave 8 co-seal batch sibling:

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [D4 Command Shaper](D4-command-shaper.md)(Wave 8 co-seal sibling) | ↔(Wave 8 co-seal)| §4.6 emission 表 + §4.9.1 D4 hand-off — `FMS_ModeMgr_to_Shape_Bus.{active_mode, ctrl_mode_hint, route_*, enable_*, bumpless_snapshot_*}` 字段值约定;每个 chart 状态 entry 上的 routing 指令 |
| [D5 多旋翼 leaf](D5-multicopter-leaf.md)(Wave 9)| → 强前置 | §4.9.4 — chart 拓扑 + 转换 + 守卫全 shared;leaf 仅注入 PARAM 与 `landed_detected` 谓词数值;ACRO 启用决策 |
| [D6 FMS↔Controller 接口](D6-fms-controller-interface.md)(Wave 8 co-seal sibling)| ↔ | §4.6 emission 表满足 D6 §4.6.1 列头与契约形状;每行 ∈ D6 §4.4.1 合法 truth-table;reset 行 = 全 0(D6 §4.7);M-11 OFFBOARD 行经 D6 §4.4 互斥过滤 |
| [E1 Controller 功能](../E-controller/E1-controller-functional.md)(Wave 8 co-seal sibling)| ⇢(经 `FMS_Out_Bus`)| §4.2 ctrl_mode 字段映射 + §4.6 cmd_mask emission 表 → E1 §4.2 trim rules 输入(D3 不约束 E1 内部算法)|
| [F3 INS 消费规则](../F-ins-contract/F3-consumption-rules.md)(Wave 9)| ⇢ | §4.5 G-ALT / G-POS / G-INS-READY 守卫 + §4.7.4 INS-validity-lost emission 政策 → F3 INS 字段消费矩阵的 Mode Manager 部分;§4.4 `ins_status_change` event |
| [H1 验证场景目录](../H-verification/H1-scenario-catalog.md)(Wave 11)| ⇢ | §4.3.2 24 条 ARM_DOMAIN 转换 + §4.3.3 9 条 FAILSAFE_OVERLAY 转换 = H1 mode-transition 用例候选;§4.5 守卫 = H1 边界用例(arm 拒绝 / INS 不就绪 / mission 未加载等)|
| [H4 故障注入目录](../H-verification/H4-fault-catalog.md)(Wave 11)| ⇢ | §4.3.3 F-01..F-09 transition + §4.4 ins_*_valid_lost / safety_failsafe_request event = H4 故障注入触发的 Mode Manager 响应剖面 |

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-08 | Wave 8 D3 author | 初稿 — chart 顶层 = 2 AND-state region(ARM_DOMAIN + FAILSAFE_OVERLAY);ARMED super 含 12 mode + DISARMED super 含 INIT + STANDBY;FAILSAFE_OVERLAY 含 NORMAL + FS-01..FS-04 AND-state overlay;§4.2 22-row state↔6-enum 映射表;§4.3 24 ARM_DOMAIN + 9 OVERLAY 转换;§4.4 22 events;§4.5 12 守卫;§4.6 16-row cmd_mask emission 表(满足 D6 §4.4.1 合法 truth-table + §4.6.1 列头);§4.7 reset/init + §4.8 overlay AND-state 论证 + §4.9 D4/D6/E1/D5 hand-off |
| 2026-05-08 | fix-up author | Wave 8 reviewer changes-requested 修复:Issue 1 — D3 §4.6 cmd_mask emission 表对 7 个 POS-driven modes(M-05 POSHOLD / M-06 TAKEOFF / M-07 LAND / M-08 RTL / M-09 LOITER / M-10 MISSION / FS-01 RTL_FAILSAFE / FS-02 LAND_NOW / FS-03 HOVER_HOLD)按 D4 §4.2.1 / §4.2.2 canonical baseline 改为 BIT_ATTITUDE_LOOP=0(姿态由 thrust-vector cascade 隐式推导,per D6 §4.3.3 / §4.3.6);契约形状段同步更新合法 truth-table 引用集合 |

## Self-check

- [x] frontmatter 完整,字段值合法
- [x] 上游文档全部存在且 status ≥ reviewed,**或**为本工作项所在 co-seal batch 的同批兄弟(架构 v1 / A1..A8 / B1..B3 / D1 / D2 已 reviewed;**D4 / D6 / E1 是 Wave 8 (D3, D4, D6, E1) co-seal batch 同批兄弟,允许在 draft 状态相互引用 — 已在 §3 依赖表标 "(co-seal Wave 8 batch sibling)"** per [RULES §6](../RULES.md))
- [x] 退出条件逐条复核完成,每条均给出依据(§6 表 7 行 + 额外覆盖段 — VehicleExtState / FlightMode / FailsafeState 映射 + cmd_mask emission)
- [x] 引用路径全部可点击访问(架构 v1 / A1..A8 / B1..B3 / D1 / D2 / D4 / D5 / D6 / E1 / E2 / F3 / H1 / H4 全用相对路径)
- [x] 不存在 RULES §5 禁则中的内容(无 .slx 截图;无 .m 可执行代码;Stateflow 用 ASCII + 表格描述;无 firmware 实现细节复述;无 PR/branch 名;不重复定义 bus/enum/param 字段表 — 字段类别引用 D2 / B1 / B2 / B3)
- [N/A] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记 — `contract_impact: no`(chart 内部状态 + 转换 + 守卫全是模型仓侧,bus/enum/param 已在 B 区拥有),N/A
- [N/A] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径 — 本文件不镜像 firmware 契约,所有契约引用走 B1/B2/B3;§3 依赖表保留 `<pending hash>` 占位以备 OQ4 / D5 leaf 后续验证使用,N/A
- [x] 下游影响已沿关系图识别完毕(§7 表覆盖 [01 §4.4](../01-design-relationships.md) D3 出边 D4 / D5 / D6 + Wave 8 co-seal sibling E1 + 衍生 F3 / H1 / H4)
- [x] 文档不超出本工作项范围(无越权设计 — 不锁 cmd_mask 位语义 / 不锁整形公式 / 不锁 controller 环路 trim / 不锁多旋翼 leaf 数值 / 不重定义 B1/B2/B3 字段)
