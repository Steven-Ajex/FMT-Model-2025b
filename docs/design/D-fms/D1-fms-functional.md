---
work_item: D1
title: FMS 功能设计
upstream: [架构v1, A1, A2, A3, A4, A5, A6, A7, A8, B1, B2, B3]
contract_impact: no
status: draft
authored_at: 2026-05-08
last_reviewed_at:
reviewer_verdict: none
---

# D1 FMS 功能设计

## 1. 目的

定义 FMS(Flight Management System)**作为整体所做之事**的系统行为契约:其负责的 mode 集合、命令源仲裁、setpoint 整形策略、安全/失效触发条件、任务范围(Phase 2 多旋翼闭环切片必需子集),并对架构 v1 §17 OQ4(`Control_Out_Bus` 是否长期作为 FMS 输入)与 OQ2(保留 vs 重写当前 FMS 结构)给出**功能层裁决**。本文件**不**做结构分解(D2)、Stateflow 详细(D3)、Command Shaper 公式(D4)、多旋翼 leaf(D5)、`cmd_mask` 位语义(D6/E1 co-seal Wave 8)。

## 2. 范围

**在范围:**

- Phase 2(多旋翼)FMS 处理的 mode 集合及其与契约 enum(B2 §4.4.4 / 4.4.5 / 4.4.6 / 4.4.7 / 4.4.8)的对应
- 命令源类别(`Pilot_Cmd_Bus` / `GCS_Cmd_Bus` / `Auto_Cmd_Bus` / `Mission_Data_Bus` / `INS_Out_Bus` / 可选 `Control_Out_Bus`)的仲裁与冲突解决策略
- 命令整形策略类别(rate/jerk 限制、stick 死区/expo、mode-条件 setpoint 覆盖、起飞/降落 profile)
- 安全/失效触发条件类别(链路丢失、估计器失效、电量、地理围栏、mode 前提缺失)及对应 mode 退化
- 任务范围(Phase 2 = waypoint 顺序 + waypoint loiter,显式排除复杂事件)
- `FMS_Out_Bus` 功能性字段 inventory(引用 A3 §4.4.6 / B1 §4.7,**不**重定义)
- OQ4 / OQ2 的功能层裁决与退出计划

**不在范围(由其他工作项处理):**

- FMS 子模块边界与内部信号 — 由 [D2 FMS 结构设计](D2-fms-structural.md) 处理
- Mode Manager Stateflow 状态/转换/事件/守卫 — 由 [D3 FMS Mode Manager 详细设计](D3-mode-manager.md) 处理
- 各整形器具体公式(rate-limiter 数学、起飞 profile 数值曲线) — 由 [D4 FMS Command Shaper 详细设计](D4-command-shaper.md) 处理
- 多旋翼特定 mode-cmd 映射、起飞/降落 profile 数值参数 — 由 [D5 多旋翼 leaf](D5-multicopter-leaf.md) 处理
- `cmd_mask` 每位**语义**(置位时控制器做什么) — 由 [D6 FMS↔Controller 接口约定](D6-fms-controller-interface.md) ↔ [E1 Controller 功能设计](../E-controller/E1-controller-functional.md) co-seal Wave 8 处理
- bus / enum / parameter 字段级 schema — 由 [B1](../B-contracts/B1-bus-inventory.md) / [B2](../B-contracts/B2-enum-inventory.md) / [B3](../B-contracts/B3-parameter-schema.md) 拥有
- 安全触发的具体阈值数值 — 已在 [B3 §4.4](../B-contracts/B3-parameter-schema.md) `FMS_PARAM` 内锁定,本文件仅给类别
- INS 字段 fallback 行为 — 由 [F3 INS 消费规则](../F-ins-contract/F3-consumption-rules.md) 处理

## 3. 依赖

| 上游 | 引用位置 | 用途 |
|---|---|---|
| 架构 v1 §7.1 / §8.1 / §12.1 / §13 / §17 OQ2,OQ4 | [架构 v1](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | FMS 模块定义、boundary、6 子模块结构、20 ms 周期、OQ2/OQ4 缺口 |
| A1 v1 评审 — §5.1.4 / §5.1.5 / §7.2 / §8.3 / §17 OQ4+OQ2 | [A1 评审](../A-architecture/A1-v1-review.md) | A1 已把 OQ4 功能裁决与 OQ2 保留/重写 stance 指派给 D1 |
| A2 命名与单位约定 | [A2](../A-architecture/A2-naming-conventions.md) | mode/source/shaper 字段名命名 + 单位/坐标系记法 |
| A3 模块边界 — §4.4(FMS bus 边界)、§4.6(passthrough 集)、§4.10(OQ4 bus 边界级处置) | [A3](../A-architecture/A3-module-boundaries.md) | FMS 输入/输出 bus 字段级清单(本文件**只引用,不重定义**);OQ4 的 bus 边界已由 A3 给出,D1 在功能层裁决 |
| A4 跨速率边界 | [A4](../A-architecture/A4-rate-boundaries.md) | RB-04(INS→FMS 10→20 ms)/ RB-05(FMS→Ctrl 20→5 ms)/ RB-06(external→FMS) |
| A5 变体策略 | [A5](../A-architecture/A5-variant-strategy.md) | Phase 2 = 多旋翼;mode 集合受变体范围限制 |
| A6 init/reset 状态契约 — §4.4.2(`FMS_Out_Bus` reset 类别)/§4.5.2(global vs local reset)/§4.5.3(INS 协调) | [A6](../A-architecture/A6-init-reset-contract.md) | mode-at-reset 行为;INS readiness gate;global/local reset 划分 |
| A7 时间约定 | [A7](../A-architecture/A7-time-conventions.md) | mode 计时(stick dwell、failsafe timeout)以 step interface timestamp 为单源 |
| A8 共享库块 — `FMS_Command_Shaper_Shell` / `Stale_Detector` / `Validity_AND` | [A8](../A-architecture/A8-shared-library-roster.md) | shaping / staleness / validity 复用块 |
| B1 Bus 清单 — §4.2(FMS 输入 bus)/§4.7(`FMS_Out_Bus` ledger) | [B1](../B-contracts/B1-bus-inventory.md) | FMS 输入/输出字段级 schema(D1 引用) |
| B2 Enum 清单 — §4.4.1..4.4.14 | [B2](../B-contracts/B2-enum-inventory.md) | `VehicleStatus` / `VehicleState` / `VehicleExtState` / `PilotMode` / `FlightMode` / `CtrlMode` / `FailsafeState` / `MissionState` / `ErrorCode` / `GCS_CmdType` / `INS_Status` / `INS_Flag` / `cmd_mask` 位号 |
| B3 Parameter schema — §4.4 `FMS_PARAM` 43 字段 | [B3](../B-contracts/B3-parameter-schema.md) | mode 阈值、takeoff/landing profile、geofence、failsafe、mission tunable;**D1 不重述数值,仅引用字段名** |
| FMT-Firmware @ `<pending hash>` | `FMT-Firmware/src/model/fms/<vehicle>/lib/FMS_types.h` | 契约镜像源(A1 §4.5.3 OQ2 stance 涉及对当前 firmware 行为的对齐) |

环境注:FMT-Firmware 在本设计阶段**未挂载**;本文件以"功能描述 + 引用 A3 / B1 / B2 / B3 已锁定字段名"形式撰写,不依赖 firmware 文件直接读取。

## 4. 设计内容

### 4.1 Mode 集合(Phase 2 多旋翼)

D1 在 Phase 2 启用以下 FMS mode。**Mode 名称是工作名**;实际 `FMS_Out_Bus.mode` 字段值由 [B2 §4.4.5 `FlightMode`](../B-contracts/B2-enum-inventory.md) 锁定 enum 数值,本节给"功能 → enum"映射。

#### 4.1.1 Mode 总表

| # | Mode 工作名 | `FlightMode` enum(B2 §4.4.5) | `CtrlMode` echo(B2 §4.4.6) | 主导命令源(§4.2) | `cmd_mask` 工作位(B2 §4.4.14;**位语义 D6/E1**) | 触发条件(摘要) |
|---|---|---|---|---|---|---|
| M-01 | DISARMED / IDLE | `MODE_NONE` | `CMODE_DISARMED` | (none) | 全 0(safe state per A6 §4.4.2) | 上电默认;disarm 命令 |
| M-02 | MANUAL | `MODE_MANUAL` | `CMODE_MANUAL_RATE` 或 `CMODE_THROTTLE_PASSTHROUGH`(由具体 sub-variant) | Pilot | `RATE_LOOP` + `THROTTLE_PASSTHROUGH`(工作名;D6/E1 锁) | Pilot `mode_switch=PMODE_MANUAL` |
| M-03 | STABILIZE | `MODE_STABILIZE` | `CMODE_MANUAL_ANGLE` | Pilot | `ATTITUDE_LOOP` + `RATE_LOOP` + `THROTTLE_PASSTHROUGH` | Pilot `mode_switch=PMODE_STABILIZE` |
| M-04 | ALTHOLD | `MODE_ALTHOLD` | `CMODE_AUTO_VEL`(垂直)+ `CMODE_MANUAL_ANGLE`(水平) | Pilot(横向 stick → 角度;垂向 stick → 垂直速度) | `ATTITUDE_LOOP` + `RATE_LOOP` + `VELOCITY_LOOP`(仅 Z) | Pilot `mode_switch=PMODE_ALTHOLD`;前提 `INS_Flag.ATTITUDE_VALID` |
| M-05 | POSHOLD | `MODE_POSHOLD` | `CMODE_AUTO_POS` | Pilot(stick = 速度偏置)+ INS(pos hold) | `POSITION_LOOP` + `VELOCITY_LOOP` + `ATTITUDE_LOOP` + `RATE_LOOP` + `YAW_LOOP` | Pilot `mode_switch=PMODE_POSHOLD`;前提 `INS_Flag.POSITION_VALID` + `VELOCITY_VALID` |
| M-06 | TAKEOFF | `MODE_NONE` 派生 *(B1 verify 是否独立 enum)* + `VehicleExtState=EXT_STATE_TAKING_OFF` | `CMODE_AUTO_POS` | Auto-internal(由 D1 起飞 profile 触发) | `POSITION_LOOP` + `VELOCITY_LOOP` + `ATTITUDE_LOOP` + `RATE_LOOP` | GCS `CMDTYPE_TAKEOFF` 或 Pilot `PMODE_MISSION` 起始;前提 ARM + INS pos valid |
| M-07 | LAND | `MODE_LAND` | `CMODE_AUTO_VEL`(垂直)| Auto-internal(降落 profile) | `VELOCITY_LOOP`(下降 vz)+ `ATTITUDE_LOOP` + `RATE_LOOP`;水平由 POSHOLD 维持 | GCS `CMDTYPE_LAND`;Pilot `PMODE_LAND`;低电 critical → land |
| M-08 | RTL | `MODE_RTL` | `CMODE_AUTO_POS` | Auto-internal(序列:climb → cruise to home → loiter → land) | 同 POSHOLD + 序列控制 | GCS `CMDTYPE_RTL`;Pilot `PMODE_RTL`;link-loss / low-bat failsafe → RTL |
| M-09 | LOITER(航点) | `MODE_POSHOLD` 复用(`MissionState=ACTIVE` 期间停在 wp) | `CMODE_AUTO_POS` | Mission(Mission Manager 决定 wp 停留时长) | 同 POSHOLD | mission progress 在某 wp `wp_acceptance_radius_m` 内且 wp 含 loiter 指令 |
| M-10 | MISSION | `MODE_MISSION` | `CMODE_AUTO_POS` | Mission(`Mission_Data_Bus.wp_array[]`)+ Auto | `POSITION_LOOP` + `VELOCITY_LOOP` + `ATTITUDE_LOOP` + `RATE_LOOP` + `YAW_LOOP` | Pilot `PMODE_MISSION` 或 GCS `CMDTYPE_RESUME_MISSION`;前提 mission loaded + INS pos/att valid |
| M-11 | OFFBOARD | `MODE_OFFBOARD` | per `Auto_Cmd_Bus.cmd_mask_request`(由 D1 仲裁后采纳)| Auto(`Auto_Cmd_Bus`) | per Auto-decided mask | Pilot `PMODE_OFFBOARD` 或 GCS `CMDTYPE_OFFBOARD`;Offboard 流活跃 |
| M-12 | ACRO | `MODE_ACRO` | `CMODE_MANUAL_RATE` | Pilot(stick → ang_rate_cmd) | `RATE_LOOP` + `THROTTLE_PASSTHROUGH` | Pilot `mode_switch=PMODE_*` 含 ACRO 选项(B2 `PMODE_*` 当前 9 项无 ACRO 单独成员;若 D5 / firmware 引入,本行随之激活;否则 ACRO 走 `PMODE_MANUAL` 子分支)|

**Failsafe-derived sub-modes(由 §4.4 Safety 触发;不是独立 mode,而是 mode 降级)**:

| # | 子工作名 | 通过修改 `mode` + `failsafe_state`(B2 `FailsafeState`)实现 | 等价 mode | 触发摘要 |
|---|---|---|---|---|
| FS-01 | RTL(failsafe) | `mode=MODE_RTL` + `failsafe_state=FAIL_RC_LOSS / FAIL_GCS_LOSS / FAIL_LOW_BATTERY / FAIL_GEOFENCE` | M-08 RTL | 失效触发自动 RTL |
| FS-02 | LAND-NOW | `mode=MODE_LAND` + `failsafe_state=FAIL_LOW_BATTERY`(critical) | M-07 LAND | critical 电量;高度过低无法 RTL |
| FS-03 | HOVER-HOLD | `mode=MODE_POSHOLD` + `failsafe_state=FAIL_GCS_LOSS / FAIL_OFFBOARD_LOSS` | M-05 POSHOLD | 等待恢复;PARAM `failsafe_gcs_loss_action=FA_HOLD`(B3 FMS_PARAM.21) |
| FS-04 | DISARM(forced) | `status=STATUS_DISARM` + `state=STATE_LOCKDOWN` + `failsafe_state=FAIL_INS_INVALID` 等 | M-01 DISARMED | 不可恢复失效;`error_code=ERR_INS_INVALID` 等 |

#### 4.1.2 Mode 数量统计

- **主 mode**:12 项(M-01 至 M-12)
- **Failsafe 派生 sub-mode**:4 项(通过修改 `failsafe_state` + mode 实现,非独立 enum 数值)
- **覆盖的 B2 `FlightMode` enum 成员**:`MODE_NONE`、`MODE_MANUAL`、`MODE_STABILIZE`、`MODE_ALTHOLD`、`MODE_POSHOLD`、`MODE_MISSION`、`MODE_RTL`、`MODE_LAND`、`MODE_OFFBOARD`、`MODE_ACRO`(B2 §4.4.5 共 10 成员,Phase 2 全集启用;`MODE_NONE` 同时承载 IDLE + TAKEOFF 派生)
- **B2 `VehicleStatus` 成员**:`STATUS_DISARM` / `STATUS_STANDBY` / `STATUS_ARM` 全启用
- **B2 `VehicleState` 成员**:`STATE_DISARM` / `STATE_STANDBY` / `STATE_LOCKDOWN` / `STATE_INAIR` / `STATE_GROUND` 全启用
- **B2 `VehicleExtState` 成员**:`EXT_STATE_LANDED` / `_TAKING_OFF` / `_HOLDING` / `_MOVING` / `_RETURNING` / `_LANDING` 全启用

#### 4.1.3 Mode-at-reset 行为(per A6)

per [A6 §4.4.2](../A-architecture/A6-init-reset-contract.md) `FMS_Out_Bus` reset 类别表:

- `mode` reset 后 = `MODE_NONE`(HARDCODED;A6 §4.4.2 行 "mode | HARDCODED(disarmed/manual/idle 中由 D3 选定)";D1 选定 `MODE_NONE`,enum 数值由 B2 §4.4.5 锁 = 0)
- `status` reset 后 = `STATUS_DISARM`
- `state` reset 后 = `STATE_DISARM`
- `ext_state` reset 后 = `EXT_STATE_LANDED`
- `ctrl_mode` reset 后 = `CMODE_DISARMED`
- `cmd_mask` reset 后 = 全 0(safe state)
- `failsafe_state` reset 后 = `FAIL_NONE`
- `error_code` reset 后 = `ERR_NONE`

**FMS init 后第一次 step 之前**,FMS 处于 M-01(DISARMED / IDLE)。**离开 M-01 的最早时机**:`INS_Flag.bit.ready==1` 且 Pilot `arm_switch==1` 且 `arm_throttle_thr_n01` (FMS_PARAM.02) 守卫满足 — 此守卫由 D3 状态机落地。

#### 4.1.4 Mode 转换矩阵(类别层 — 行→列允许)

D1 给**类别层**允许性,**精确转换图**由 D3 Stateflow 锁定。

| from \ to | M-01 IDLE | M-02 MANUAL | M-03 STABILIZE | M-04 ALTHOLD | M-05 POSHOLD | M-06 TAKEOFF | M-07 LAND | M-08 RTL | M-09 LOITER | M-10 MISSION | M-11 OFFBOARD | M-12 ACRO |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| M-01 IDLE | — | arm | arm | arm(+INS) | arm(+INS) | GCS+arm+INS | — | — | — | — | — | arm |
| M-02 MANUAL | disarm | — | pilot | pilot+INS | pilot+INS | — | — | failsafe | — | — | — | pilot |
| M-03 STABILIZE | disarm | pilot | — | pilot+INS | pilot+INS | — | pilot | failsafe | — | — | — | pilot |
| M-04 ALTHOLD | disarm | pilot | pilot | — | pilot+INSpos | — | pilot | failsafe | — | — | — | pilot |
| M-05 POSHOLD | disarm(land) | pilot | pilot | pilot | — | — | pilot/auto | failsafe/pilot | mission | mission | offboard | pilot |
| M-06 TAKEOFF | failsafe | — | — | — | auto→ | — | abort | failsafe | — | auto→ | — | — |
| M-07 LAND | auto→disarm(touchdown) | — | — | — | abort/pilot | — | — | failsafe | — | — | — | — |
| M-08 RTL | auto→land→disarm | — | — | — | pilot abort | — | auto→land | — | wp_arrived | — | — | — |
| M-09 LOITER | failsafe | — | — | — | timeout | — | pilot | failsafe | — | mission resume | — | — |
| M-10 MISSION | failsafe | — | — | — | pilot pause | — | mission end | failsafe/end | wp loiter | — | — | — |
| M-11 OFFBOARD | failsafe | pilot | pilot | pilot | offboard end | — | offboard land | failsafe | — | — | — | pilot |
| M-12 ACRO | disarm | pilot | pilot | — | — | — | — | failsafe | — | — | — | — |

**说明**:`pilot` = pilot mode_switch 触发;`auto→` = D1 内部序列推进;`failsafe` = §4.4 触发条件命中;`+INS` = 需 INS gate 满足;`disarm` = 触地或 GCS disarm。"—" 表示 D1 类别层禁止;D3 Stateflow 在自身退出条件中复核每条转换。

### 4.2 命令源(Command Sources)

per 架构 v1 §8.1 + A3 §4.4.1,FMS 仲裁以下命令源:

#### 4.2.1 命令源清单

| Source | bus | 类别 | 在哪些 mode 中主导 | A3 引用 |
|---|---|---|---|---|
| **PILOT** | `Pilot_Cmd_Bus` | mandatory(契约 + 功能;Phase 2 闭环必需) | M-02..M-05、M-12;在 M-06..M-11 中作为"pilot intervention" override 来源 | A3 §4.4.2 |
| **GCS** | `GCS_Cmd_Bus` | optional 功能(契约 mandatory) | 模式切换命令(`CMDTYPE_MODE_CHANGE`)、起飞/降落/RTL 触发(`CMDTYPE_TAKEOFF` / `_LAND` / `_RTL`)、setpoint 更新(`_SETPOINT_UPDATE`)、set home(`_SET_HOME`)、mission 控制(`_PAUSE_MISSION` / `_RESUME_MISSION`) | A3 §4.4.3.1 |
| **AUTO** | `Auto_Cmd_Bus` | optional | 仅 M-11 OFFBOARD 主导;M-06 / M-07 / M-08 期间由 D1 内部 "auto" 状态生成 setpoint(不消费 `Auto_Cmd_Bus`,称为 internal-auto) | A3 §4.4.3.2 |
| **MISSION** | `Mission_Data_Bus` | optional | M-09 / M-10 主导;M-08 RTL 部分使用(home as implicit waypoint) | A3 §4.4.3.3 |
| **NAV(state)** | `INS_Out_Bus` | mandatory 功能 + 契约 | 所有 mode 都消费(状态反馈、validity gate);per A3 §4.4.4 | A3 §4.4.4 |
| **CONTROL(echo)** | `Control_Out_Bus` | optional / **transitional**(per OQ4 — §4.7) | 仅诊断 / 安全 cross-check(controller-zero-output detection);**不**作为 setpoint 来源 | A3 §4.4.5 / §4.10 |

#### 4.2.2 仲裁优先级(冲突解决)

源-级冲突(同一 mode 多源同时给出意图)按以下优先级,**高 → 低**:

1. **SAFETY override**(由 §4.4 触发):任何 failsafe 命中时,Safety Monitor 直接命令 Mode Manager 切到对应 FS-* 子 mode,**override 所有其他源的 mode 请求**(per 架构 v1 §12.1.3 "can request mode degradation through the Mode Manager only")。
2. **PILOT mode_switch**(`Pilot_Cmd_Bus.mode_switch` 变化):pilot 始终保留 mode-switch 权;若 pilot 切到 `PMODE_MANUAL` / `PMODE_STABILIZE` / `PMODE_ALTHOLD` / `PMODE_POSHOLD` / `PMODE_RTL` / `PMODE_LAND`,**优先于** GCS / Auto / Mission 的 mode 请求(pilot intervention)。
3. **PILOT kill_switch / arm_switch off**:立即触发 FS-04 DISARM(等同 SAFETY override)。
4. **GCS `CMDTYPE_MODE_CHANGE`**:在 pilot 未变 mode_switch 时由 GCS 决定 mode。
5. **AUTO `cmd_mask_request`**(M-11 OFFBOARD 期间):FMS 决定是否采纳(D6 cmd_mask 语义层裁决;D1 仅给"在 M-11 中 Auto 的请求**可能**被采纳"原则)。
6. **MISSION wp 推进**(M-09 / M-10 期间):mission manager 决定 setpoint 来源;pilot stick 在 mission 模式下默认**忽略 roll/pitch/throttle**,但 **honor yaw stick**(yaw intervention,per §4.3 整形规则)。

#### 4.2.3 同源内冲突(staleness / validity)

- **Staleness**:任何源 bus 的 `timestamp` 与当前 step interface timestamp(per A7)差值 > 对应 PARAM(`link_loss_timeout_s` FMS_PARAM.16 / `gcs_loss_timeout_s` FMS_PARAM.17)即视为 stale → `valid` 字段 effectively false → 命中 §4.4 失效触发。staleness 检测通过 [A8 §4.2.5 `Stale_Detector`](../A-architecture/A8-shared-library-roster.md) 复用块实现。
- **Validity**:每个源的 `valid` 字段 = false 时,该源在仲裁中**视为不存在**(不输入设定点,但 SAFETY 触发可能命中)。
- **INS validity**:per [A3 §4.4.4 / §4.8](../A-architecture/A3-module-boundaries.md) 矩阵,INS gate 由各子模块按位检查(详见 §4.4 INS-derived 触发)。

#### 4.2.4 命令源至 mode 的"喂养"映射

per A3 §4.4.6.1 setpoint 字段产出:

| Mode | pos_cmd 来源 | vel_cmd 来源 | acc_cmd 来源 | att/yaw_cmd 来源 | rate_cmd 来源 | throttle_cmd 来源 |
|---|---|---|---|---|---|---|
| M-01 IDLE | ZERO | ZERO | ZERO | ZERO | ZERO | ZERO |
| M-02 MANUAL | — (cmd_mask off) | — | — | — | Pilot stick × shaper(D4) | Pilot stick passthrough(per A3 §4.4.6.4) |
| M-03 STABILIZE | — | — | — | Pilot stick → angle(D4) | (内环由 Controller 计算) | Pilot stick passthrough |
| M-04 ALTHOLD | — | Pilot stick(z 轴)→ vz | — | Pilot stick(横向)→ angle | (内环) | (Controller 闭环垂向) |
| M-05 POSHOLD | INS pos hold(stick=0)/ pilot stick offset | Pilot stick → vel offset | shaper jerk-limited | yaw: pilot stick × shaper | (内环) | (闭环) |
| M-06 TAKEOFF | home + `takeoff_alt_m`(FMS_PARAM.06)| `takeoff_climb_speed_mps`(.07) | shaper jerk-limited | yaw: hold init | (内环) | (闭环) |
| M-07 LAND | hold horizontal | `landing_descent_speed_mps`(.08)→ `land_touchdown_speed_mps`(.09) flare | shaper | yaw: hold | (内环) | (闭环) |
| M-08 RTL | home + `rtl_climb_alt_m`(.28)→ home → land seq | derived | shaper | yaw: face home | (内环) | (闭环) |
| M-09 LOITER | wp_current | 0(hold)| ZERO | yaw: wp 指定 / hold | (内环) | (闭环) |
| M-10 MISSION | wp_current → next | `mission_default_speed_mps`(.27) | shaper | yaw: 飞行方向 / wp 指定 | (内环) | (闭环) |
| M-11 OFFBOARD | `Auto_Cmd_Bus.target_pos_ned_m` | `target_vel_ned_mps` | `target_acc_ned_mps2` | `target_yaw_rad` / `target_yaw_rate_radps` | (内环) | (闭环) |
| M-12 ACRO | — | — | — | — | Pilot stick × shaper(D4) | Pilot stick passthrough |

`—` 表示该字段由 `cmd_mask` 关闭,字段值 = ZERO。具体哪一位 `cmd_mask` 关闭由 D6/E1 锁定。

### 4.3 命令整形策略(Command Shaping — 功能层)

D1 给**整形类别**;具体公式由 [D4 Command Shaper](D4-command-shaper.md) 落地。所有整形复用 [A8 §4.3.2 `FMS_Command_Shaper_Shell`](../A-architecture/A8-shared-library-roster.md) 库块。

#### 4.3.1 RC stick 输入预处理

| 整形项 | 适用源/字段 | PARAM(B3) | 说明 |
|---|---|---|---|
| 死区(deadband) | `Pilot_Cmd_Bus.{roll,pitch,yaw,throttle}_stick_n01` | `manual_stick_deadzone_n01`(FMS_PARAM.40)| stick 在 ±dz 内视为 0 |
| Expo(指数曲线) | 同上(roll/pitch/yaw 杆位) | TBD by D4(Phase 2 默认线性,可选 expo) | 提升小杆位精细度 |
| 单位换算 | stick n01 → 物理单位(rad / rad·s⁻¹ / m·s⁻¹ / 0..1)| 由 `tilt_lim_rad`(FMS_PARAM.38)/ `yaw_rate_lim_radps`(.39)/ `vel_xy_lim_mps`(.33)等映射上限 | mode-条件:不同 mode 下 stick 映射目标不同(§4.2.4) |
| Stick 静态有效性 | `arm_switch` / `kill_switch` / `valid` | per A6 §4.5.3 守卫 | reset 后 HARDCODED false,直至 source 注入 |

#### 4.3.2 Setpoint rate / jerk 限制

| 限制项 | 字段(per A3 §4.4.6.1) | PARAM | 说明 |
|---|---|---|---|
| 水平速度上限 | `vel_cmd_ned_mps[0..1]` | `vel_xy_lim_mps`(FMS_PARAM.33) | 模值 ≤ lim |
| 垂直速度上限 | `vel_cmd_ned_mps[2]` | `vel_z_up_lim_mps`(.34) / `vel_z_down_lim_mps`(.35) | 上下 asymmetric |
| 水平加速度上限 | `acc_cmd_ned_mps2[0..1]` | `acc_xy_lim_mps2`(.36) | 模值 ≤ lim |
| 水平 jerk 上限 | (D4 内部状态;不直接落 bus 字段) | `jerk_xy_lim_mps3`(.37) | jerk-limited profile 内部约束 |
| 倾角上限 | `att_cmd_quat`(转 tilt 角)| `tilt_lim_rad`(.38) | 倾角饱和 |
| Yaw 速率上限 | `yaw_rate_cmd_radps` | `yaw_rate_lim_radps`(.39) | |
| Hover 油门(manual)| `throttle_cmd` | `manual_thr_hover_n01`(.41) | manual mode 下 stick 中位映射的油门 |

#### 4.3.3 Mode-conditional setpoint override

- **AUTO 模式 ignore RC roll/pitch**:M-06..M-10 中,Pilot `roll_stick_n01` / `pitch_stick_n01` **默认不进入** setpoint;若 D5 leaf 启用 "pilot intervention",Pilot stick 大于 deadzone 时退化到 M-05 POSHOLD(由 D3 触发)。
- **AUTO 模式 honor yaw stick**:Pilot `yaw_stick_n01` 在 AUTO 模式下作为 `yaw_rate_cmd_radps` 偏置叠加(yaw intervention),不触发 mode 退化。
- **AUTO 模式 ignore throttle stick**(在 M-04..M-10 中)。
- **MANUAL passthrough 字段**:per A3 §4.4.6.4,`actuator_cmd[]` / `pilot_throttle_passthrough` / `pilot_yaw_rate_passthrough_radps` 是 passthrough(经 D4 shaper 但语义来源是 pilot);仅在 M-02 MANUAL 启用。`motor_cmd_passthrough[]` 多旋翼 Phase 2 默认禁用(per A3 §4.4.6.4)。

#### 4.3.4 起飞 / 降落 profile 生成(类别层)

D1 仅给"profile 类别 + PARAM 引用",**数值曲线** D4 + D5 落地。

| Profile | mode | 阶段 | 主要 PARAM(B3) |
|---|---|---|---|
| TAKEOFF | M-06 | (1) lift-off detection(throttle 上升 + alt 微变)→(2) climb at `takeoff_climb_speed_mps`(.07)→(3) reach `takeoff_alt_m`(.06)+ POSHOLD lock | .06 / .07 |
| LAND | M-07 | (1) descent at `landing_descent_speed_mps`(.08)→(2) flare at low alt(高度阈值 D4/D5 锁)→(3) touchdown at `land_touchdown_speed_mps`(.09)→(4) `land_disarm_dwell_s`(.10) 后 disarm | .08 / .09 / .10 |
| RTL | M-08 | (1) climb to `rtl_climb_alt_m`(.28)→(2) cruise to home(POSHOLD-style)→(3) loiter `rtl_loiter_time_s`(.29)→(4) auto-land seq(LAND profile) | .28 / .29 + LAND PARAMs |

#### 4.3.5 共享整形原语

per [A8 §4.2 / §4.3](../A-architecture/A8-shared-library-roster.md):

- `Rate_Limiter_1d` / `Rate_Limiter_3d`:vel / acc / yaw_rate 限速
- `Jerk_Limiter_3d`:水平 jerk 约束
- `Deadzone_With_Linear_Bridge`:stick 死区 + 线性桥接(避免阶跃)
- `Validity_AND` / `Stale_Detector`:source validity + staleness gate
- `FMS_Command_Shaper_Shell`:整形包装容器,被 D4 / D5 实例化

### 4.4 安全 / 失效触发条件(Safety / Failsafe)

D1 给**触发条件类别**;具体阈值由 B3 `FMS_PARAM` 锁;每条触发的退化目标由本节 → 仲裁(§4.2.2 第 1 优先级)→ Mode Manager(D3)落地。

#### 4.4.1 触发清单

| 触发 ID | 触发条件 | 测量字段 | 阈值 PARAM(B3) | 退化目标 mode | `failsafe_state`(B2) | `error_code`(B2) |
|---|---|---|---|---|---|---|
| TR-01 RC link loss | `Pilot_Cmd_Bus.valid==0` 或 staleness > timeout | `Pilot_Cmd_Bus.{valid, timestamp}` | `link_loss_timeout_s`(.16) | per `failsafe_link_loss_action`(.20) — 默认 `FA_RTL` → M-08 RTL | `FAIL_RC_LOSS` | `ERR_RC_LOST` |
| TR-02 GCS link loss | `GCS_Cmd_Bus.valid==0` 或 staleness > timeout | `GCS_Cmd_Bus.{valid, timestamp}` | `gcs_loss_timeout_s`(.17) | per `failsafe_gcs_loss_action`(.21) — 默认 `FA_HOLD` → M-05 POSHOLD(FS-03) | `FAIL_GCS_LOSS` | (无独立 ErrorCode 成员;由 H4 决定是否新增) |
| TR-03 INS validity loss | `INS_Status != READY` 或 关键 `INS_Flag` 位连续 0 超过 timeout | `INS_Out_Bus.{INS_Status, flag}` | `ins_unready_timeout_s`(.24)(per A6 §4.5.3) | M-01 DISARMED + LOCKDOWN(FS-04);若 INAIR 时则先 M-07 LAND | `FAIL_INS_INVALID` | `ERR_INS_INVALID` |
| TR-04 GPS lost | `INS_Flag.GPS_VALID==0` (sustained) | `INS_Out_Bus.flag.gps_valid` | (用 `ins_unready_timeout_s` 子集 / TBD by D5) | 在 M-05/M-08/M-09/M-10 期间 → M-04 ALTHOLD(降级);其它 mode 不退化 | `FAIL_INS_INVALID`(子情况)| `ERR_GPS_LOST` |
| TR-05 Battery low | 外部电池监控信号 | (由 H4 故障目录决定来源;Phase 2 placeholder)| `low_battery_voltage_v`(.18) | per `failsafe_low_bat_action`(.22) — 默认 `FA_RTL` → M-08 RTL(FS-01)| `FAIL_LOW_BATTERY` | `ERR_BATTERY_LOW` |
| TR-06 Battery critical | 同上 | 同上 | `critical_battery_voltage_v`(.19) | per `failsafe_critical_bat_action`(.23) — 默认 `FA_LAND` → M-07 LAND(FS-02) | `FAIL_LOW_BATTERY`(critical 子级)| `ERR_BATTERY_CRITICAL` |
| TR-07 Geofence breach | 位置 / 高度越界 | `INS_Out_Bus.position_ned_m` + `INS_Out_Bus.position_lla.alt_m` | `geofence_enabled`(.11) / `geofence_shape`(.12) / `geofence_radius_m`(.13) / `geofence_alt_max_m`(.14) | per `geofence_action`(.15) — 默认 `FA_RTL` → M-08 RTL(FS-01) | `FAIL_GEOFENCE` | `ERR_GEOFENCE` |
| TR-08 Mode preconditions invalid | mode 切换请求但前提 INS 字段未就绪 / arm 未满足 | per mode 表(§4.1.1 触发条件列) | (gate 由 FMS_PARAM `arm_*` 系列 + INS flag) | 拒绝转换(保持当前 mode);设置 `error_code=ERR_MODE_INVALID` | `FAIL_NONE`(不是 failsafe,是拒绝转换)| `ERR_MODE_INVALID` |
| TR-09 Offboard stream loss | `Auto_Cmd_Bus.valid==0` (M-11 期间) | `Auto_Cmd_Bus.{valid, timestamp}` | `gcs_loss_timeout_s`(.17) 复用 / TBD | M-05 POSHOLD(FS-03)| `FAIL_OFFBOARD_LOSS` | `ERR_OFFBOARD_TIMEOUT` |
| TR-10 Pilot kill_switch | `Pilot_Cmd_Bus.kill_switch==1` | `Pilot_Cmd_Bus.kill_switch` | (immediate;无 dwell) | M-01 DISARMED + LOCKDOWN(FS-04) | `FAIL_NONE`(用户主动) | `ERR_NONE` |
| TR-11 Motor fault | 来自 Plant / Controller fault enum 翻译为 `ErrorCode` | (per H4 故障目录)| (由 C / E 决定阈值;FMS 仅消费翻译后的 ErrorCode) | M-07 LAND(FS-02) | `FAIL_NONE`(motor)| `ERR_MOTOR_FAULT` |

#### 4.4.2 触发优先级

同一 step 内多触发命中时,**高 → 低**:

1. TR-10 Pilot kill_switch / TR-03 INS validity loss(critical safety,`STATE_LOCKDOWN`)
2. TR-06 Battery critical / TR-11 Motor fault(强制 LAND)
3. TR-07 Geofence breach / TR-05 Battery low(RTL)
4. TR-01 RC link loss / TR-09 Offboard stream loss
5. TR-02 GCS link loss
6. TR-04 GPS lost(部分降级,非全失效)
7. TR-08 Mode preconditions invalid(仅拒绝,不退化)

冲突时:**只走最高优先级一条退化路径**;`failsafe_state` 反映该最高条;`error_code` 取相应 enum。后续触发若不变化 mode,仅在 step trace / log 中记录。

#### 4.4.3 触发的 debounce / hysteresis

- 链路 / Offboard staleness 触发:`Stale_Detector`(A8 §4.2.5)以 PARAM-stated timeout 计时,**单边触发**(staleness > threshold → trigger)+ **回滞恢复**(staleness 恢复后再保持 `failsafe_recovery_dwell_s`(FMS_PARAM.43)才允许 mode 恢复请求)。
- INS validity 触发:gate 位连续 N 个 step 为 0 才触发(N = `ins_unready_timeout_s` × 50 Hz),避免单帧抖动。
- Geofence 触发:**进入** breach 立即触发;**退出** breach 后保持 RTL 直至 D3 显式接受 pilot resume。
- Battery 触发:阈值 + 5% 回滞带(D4 / D5 锁定具体回滞数值)。

#### 4.4.4 Failsafe 退化目标的可恢复性

- **可恢复**(failsafe 解除 + dwell 后,可由 pilot 切回普通 mode):TR-01 / TR-02 / TR-04 / TR-05 / TR-09
- **不可恢复**(必须 disarm 后重新 arm):TR-03 / TR-10 / TR-11(FS-04)
- **强制完成 sequence**:TR-06(LAND 完成才能再 arm);TR-07(可由 pilot 在 RTL 完成后接管)

恢复路径具体由 D3 状态机锁定;本文件给类别。

### 4.5 任务范围(Mission scope — Phase 2 cut)

per 架构 v1 §16 + 00-design-plan §4.D D1 退出条件 "mission 范围(仅闭环切片需要的子集)" + 审计 F-30 cut。

#### 4.5.1 在范围(Phase 2 启用)

| 功能 | 落地字段 / PARAM | 备注 |
|---|---|---|
| 航点序列加载 | `Mission_Data_Bus.wp_array[]`(长度 ≤ `wp_array_max_len` FMS_PARAM.25 = 64)+ `wp_count` + `wp_current_index`(per A3 §4.4.3.3) | mission planner 一次性加载;mid-flight reload **不在 Phase 2** |
| 航点顺序推进(in/out)| Mission Manager 内部逻辑 → `FMS_Out_Bus.wp_index` 推进;到达判定 = 距离当前 wp ≤ `wp_acceptance_radius_m`(FMS_PARAM.26)| 仅"到达即推进"模式;无 trigger / event 控制 |
| 航点停留(loiter at wp)| 若 wp 含 loiter 字段 → M-09 LOITER;停留时长由 wp 字段携带或由 D5 默认 | 时长字段在 `wp_array[i].param[]` 中(A3 §4.4.3.3 wp_array struct,B1 锁字段);默认时长 D5 锁 |
| 默认任务速度 | `mission_default_speed_mps`(FMS_PARAM.27) | wp-level 速度覆盖暂不支持 Phase 2 |
| Home 设置 / RTL home 引用 | `Mission_Data_Bus.home_*` + `FMS_Out_Bus.home_*`(per A3 §4.4.6.3) | RTL 使用 home 作为隐式终 wp |
| Mission pause / resume | GCS `CMDTYPE_PAUSE_MISSION` / `_RESUME_MISSION`(B2 §4.4.13)→ `MissionState=PAUSED / ACTIVE` | mission 状态在 pause 期间 PRESERVED(per A6 §5 PRESERVED 类别) |
| Mission abort | failsafe 触发(§4.4)→ `MissionState=ABORTED` | mission 不可在同 boot 内 resume(若 abort) |

#### 4.5.2 不在 Phase 2 范围(显式排除,审计 F-30)

- 复杂 mission 事件:trigger waypoint(条件触发)、loiter-with-altitude-change、figure-8、splines 等
- Payload 触发:相机快门、撒播、抓取等 payload 命令(`Mission_Data_Bus.wp_array[i].cmd_type` 中的 payload 子类型)
- Geofence "return-and-resume":geofence 触发后无法自动恢复 mission;只能手动 resume(若 RTL 后 pilot 接管再 mission resume)
- Mid-flight mission reload:Phase 2 只支持 boot-time + ground-time 加载
- Survey / grid / mowing-pattern:任何"自动生成密集 wp 网格"的高级模式
- Multi-vehicle mission coordination
- Smart RTL(基于飞行轨迹回溯路径):Phase 2 RTL 只走"climb → cruise to home → land" 三段直线
- Geofence polygon shape:`geofence_shape` PARAM 含 `GF_POLYGON` 成员,但 D1 Phase 2 仅启用 `GF_NONE` / `GF_CYLINDER`(D5 复核)

排除项的功能由 Phase 4(per 00-design-plan §… / 架构 v1 §18 Phase 4)开启。

#### 4.5.3 Mission 子状态

per [B2 §4.4.8 `MissionState`](../B-contracts/B2-enum-inventory.md):Phase 2 启用 `MISSION_IDLE` / `_LOADED` / `_ACTIVE` / `_PAUSED` / `_COMPLETE` / `_ABORTED` 全集。该字段是否进入 `FMS_Out_Bus` 由 B1 锁定其存在性(per B2 §4.4.8 备注);若不进入,FMS 内部 enum,passive log。

### 4.6 输出 bus(`FMS_Out_Bus`)功能字段

D1 列**FMS 产出的功能字段类别**;字段级 schema 由 [A3 §4.4.6](../A-architecture/A3-module-boundaries.md) + [B1 §4.7](../B-contracts/B1-bus-inventory.md) 拥有;reset 类别由 [A6 §4.4.2](../A-architecture/A6-init-reset-contract.md) 锁定。**本节不重复字段表**,仅给"D1 视角的功能产出 inventory"。

#### 4.6.1 setpoint 组(per active mode)

| 组 | 字段(A3 §4.4.6.1) | 由哪些 mode 产出非零 | 关闭时由 `cmd_mask` 屏蔽 |
|---|---|---|---|
| 位置 | `pos_cmd_ned_m[3]` | M-05 / M-06 / M-07(水平 hold)/ M-08 / M-09 / M-10 / M-11 | 工作位 `MASK_BIT_POSITION_LOOP`(B2 §4.4.14;语义 D6/E1) |
| 速度 | `vel_cmd_ned_mps[3]` | M-04(z)/ M-05 / M-06 / M-07 / M-08 / M-10 / M-11 | `MASK_BIT_VELOCITY_LOOP` |
| 加速度 | `acc_cmd_ned_mps2[3]` | shaper jerk-limited 输出;M-05/M-06/M-08/M-10/M-11 | `MASK_BIT_ACCELERATION_LOOP` |
| 姿态(quat / euler 二选一) | `att_cmd_quat[4]` 或 `att_cmd_euler_rad[3]` | M-03 / M-04(横向)/ 闭环间接(POS→ATT cascading 由 Controller) | `MASK_BIT_ATTITUDE_LOOP` |
| 角速率 | `ang_rate_cmd_b_radps[3]` | M-02 / M-12(直传)/ Controller 内环间接 | `MASK_BIT_RATE_LOOP` |
| Yaw / yaw rate | `yaw_cmd_rad` / `yaw_rate_cmd_radps` | M-04..M-12(yaw 控制)| `MASK_BIT_YAW_LOOP` / `MASK_BIT_YAW_RATE_LOOP` |
| 油门 | `throttle_cmd` | M-02 / M-03(passthrough);M-04..M-11 闭环由 Controller(此字段 = 0) | `MASK_BIT_THROTTLE_PASSTHROUGH` |

#### 4.6.2 控制流 / 状态组

| 字段 | enum(B2)| 产出方 |
|---|---|---|
| `cmd_mask` | bit-position 工作名 per B2 §4.4.14 | D1 决定**设置哪些位**;**位语义** D6/E1 锁 |
| `ctrl_mode` | `CtrlMode`(B2 §4.4.6) | per §4.1.1 mode 表 echo |
| `mode` | `FlightMode`(B2 §4.4.5) | D3 Mode Manager 输出 |
| `status` | `VehicleStatus`(B2 §4.4.1) | D3 |
| `state` | `VehicleState`(B2 §4.4.2) | D3 |
| `ext_state` | `VehicleExtState`(B2 §4.4.3) | D3 + Mission Manager |
| `reset` *(if exists per B1)* | bool | D2 / D3 触发(global reset signal,per A6 §4.5.2) |

#### 4.6.3 任务 / 导航 / 错误组

| 字段 | 来源 | 备注 |
|---|---|---|
| `wp_index` | `Mission_Data_Bus.wp_current_index` echo | per A3 §4.4.6.3 |
| `wp_lat_deg` / `wp_lon_deg` / `wp_alt_m` | `Mission_Data_Bus.wp_array[wp_current_index]` 抽取 | passthrough-direct(A3 §4.6) |
| `home_lat_deg` / `_lon_deg` / `_alt_m` | `Mission_Data_Bus.home_*` 或 GCS `CMDTYPE_SET_HOME` | passthrough-direct(A3 §4.6);默认值 PARAM .30/.31/.32 |
| `error_code` | Safety Monitor 推出 | per §4.4 表 |
| `failsafe_state` *(optional per B1)* | Safety Monitor 推出 | per §4.4 表 |

#### 4.6.4 Passthrough 字段(per A3 §4.6)

D1 不**生成**这些字段;FMS 直传或 echo:

- `actuator_cmd[]` ← Pilot/GCS/Auto(passthrough-with-shape;M-02 主用)
- `motor_cmd_passthrough[]` ← Pilot/Auto(passthrough-direct;**Phase 2 多旋翼默认禁用**,per A3 §4.4.6.4)
- `pilot_throttle_passthrough` ← `Pilot_Cmd_Bus.throttle_stick_n01`(M-02 主用)
- `pilot_yaw_rate_passthrough_radps` ← Pilot yaw stick × shaper(各 mode 共用)
- `wp_*` / `home_*` ← Mission_Data_Bus(已在 §4.6.3)
- INS validity echo 字段(`ins_ready` / `ins_position_valid` / `ins_attitude_valid`)— 若 B1 锁定其存在则 D1 echo from `INS_Out_Bus.flag`;若不存在则 Controller 直消费 INS bus(per A3 §4.4.6.5)

#### 4.6.5 Init payload(per B3 FMS_PARAM)

per [B3 §4.4.2](../B-contracts/B3-parameter-schema.md) `init_use=yes` 字段:

- `default_mode_at_boot`(FMS_PARAM.01)→ `FMS_Out_Bus.mode` 的 init echo(注:稳态由 A6 HARDCODED 锁,此 PARAM 用于"非 disarmed 启动" — Phase 2 多旋翼默认 `PMODE_MANUAL` 但 D3 init 后实际仍走 `MODE_NONE` 直至 arm)
- `arm_safety_check_mask`(.05)→ D3 守卫
- `geofence_*`(.11..15)→ Safety Monitor init
- `failsafe_*_action`(.20..23)→ Safety Monitor init
- `home_*_default`(.30..32)→ `FMS_Out_Bus.home_*` init echo
- `wp_array_max_len`(.25)→ Mission Manager 数组维数(compile-inlined)

### 4.7 OQ4 闭合:`Control_Out_Bus` 作为 FMS 输入

闭合 [架构 v1 §17 OQ4](../../architecture/2026-05-05-fmt-model-architecture-v1.md) + [A1 §4.3 OQ4 "Closed by D1+A3"](../A-architecture/A1-v1-review.md) + [A3 §4.10](../A-architecture/A3-module-boundaries.md) 已落 bus 边界 + 审计 F-23。

#### 4.7.1 D1 裁决

**选 transitional**(A3 §4.4.5 选项 2)— Phase 2 启用 `Control_Out_Bus` 作为 FMS 输入,**仅用于 MIL parity / 安全 cross-check / 诊断 logging**,**不**作为 setpoint 来源。Phase 5+ 有计划移除该输入。

#### 4.7.2 裁决理由

1. **MIL parity**:当前 firmware FMS interface 已读 `Control_Out_Bus`(per [架构 v1 §5.1.4 行 91](../../architecture/2026-05-05-fmt-model-architecture-v1.md))。Phase 2 保留以维持 firmware drop-in 兼容,避免在初次模型仓 codegen 中触发 interface 层调整(I4 codegen 配置层调整成本)。
2. **安全 cross-check 价值**:Safety Monitor 可消费 `Control_Out_Bus.motor_cmd[]` 检测 "controller 持续输出零推力但 mode != DISARMED"(异常征兆),作为 H4 故障目录中"controller silent fault" 的检测来源(留给 H4 / D2 Safety 子模块利用)。
3. **不引入设定点回路**:D1 显式声明 FMS **不**用 `Control_Out_Bus` 任何字段产生 setpoint;反之即引入隐式回路,违反架构 v1 §7.1 "FMS does not own inner-loop stabilization"(架构 v1 §7.1 Bullet 3)。
4. **退路稳定**:per [A3 §4.10 选项 2](../A-architecture/A3-module-boundaries.md),A3 已为 transitional 选项保留 bus 边界 RB-04 等价(5 ms → 20 ms downsample);本裁决落地无需 A3 修改。
5. **OQ2 互动**:OQ2 选 "macro 保留 + leaf 重写"(§4.8),与 OQ4 transitional 一致 — macro 层维持 firmware FMS 行为(包括 `Control_Out_Bus` 输入),leaf 层重新实现允许 Phase 5+ 在不破坏 macro 契约下移除该输入。

#### 4.7.3 退出计划

| Phase | 动作 | 触发条件 |
|---|---|---|
| Phase 2(本)| 启用 `Control_Out_Bus` 作为 FMS 输入(optional / functional);Safety Monitor + diagnostic logging 可消费;**不**派生 setpoint | 当前 |
| Phase 3 | 评估 SIH 中 firmware 是否真依赖 `Control_Out_Bus` 回送 → 若否,标 deprecated(`@deprecated since Phase 3` 注释,字段保留)| Phase 3 SIH 启动后 |
| Phase 5 | 移除 `Control_Out_Bus` from FMS interface 函数签名 + I4 codegen 配置消除该 input port + interface 层调整 | (a) Phase 4 完成多旋翼功能完备;(b) firmware 侧确认无依赖;(c) I4 codegen 配置项支持移除 |

退出动作的实际执行由后续工作项(Phase 5 时新增的 D-area 工作项 / I4 codegen 修订)负责;**D1 在本节登记退出计划但不执行移除**。若 OQ2 后续从 "transitional" 升级到 "永久启用"(A3 §4.10 选项 1),由 D1 走 §8 变更日志 + INDEX 决策日志同步。

### 4.8 OQ2 闭合:保留现有 FMS 结构 vs 重写

闭合 [架构 v1 §17 OQ2](../../architecture/2026-05-05-fmt-model-architecture-v1.md) + [A1 §4.5.3 / §17 OQ2](../A-architecture/A1-v1-review.md) + 审计 F-26。

#### 4.8.1 D1 裁决

**选 "macro 保留 + leaf 重写"**:

- **Macro 层(6 子模块结构)保留**:Source Selector / Mode Manager / Safety Monitor / Mission Manager / Command Shaper / Output Assembler 六块拓扑(per [架构 v1 §12.1.1..12.1.6](../../architecture/2026-05-05-fmt-model-architecture-v1.md))**完全沿用**当前 firmware FMS 的"概念分层"
- **Leaf 层(Stateflow / 整形器 / 多旋翼 leaf)重写**:每个子模块的内部 Stateflow / Simulink 块 / 整形公式 **重新撰写**,以对齐新 bus 契约(B1)+ 新 enum(B2)+ 新 PARAM(B3)+ 新 reset 协议(A6)+ 新 cmd_mask 编排(D6/E1 Wave 8 锁)
- **mode 集合**(§4.1)和**功能映射**(§4.2 / §4.3 / §4.4 / §4.5)与当前 firmware FMS 保持**功能等价**(每条 mode / source / shaper / failsafe 与 firmware 中现有可识别行为对齐),但**实现不同**

#### 4.8.2 裁决理由

1. **Macro 保留的价值**:6 子模块结构在架构 v1 §12.1 已被审查通过;它对应于"职责清晰、跨变体复用"的工程边界;若推翻重新切分会触发 D2 重新审议(成本高、收益不明确)。
2. **Leaf 重写的必要性**:
   - 新 bus 契约(B1)字段名 / 类型与 firmware 当前可能不完全一致(B1 §4.7 字段顺序 ledger 显式标 "B1 to confirm" 多处);沿用 firmware leaf 会引入 byte-level diff(B4)
   - 新 PARAM(B3)43 字段是**重新设计**(B3 §4.4 标 init 稿);firmware 现有 FMS_PARAM 字段集 / 顺序 / 默认 与 B3 期望存在差异 → leaf 必须按 B3 重新接线
   - reset 协议(A6 §4.4.2 类别表)是**新约定**;firmware FMS 当前 reset 行为可能不严格满足"所有字段 reset 类别" — leaf 必须重写以满足
   - cmd_mask 位编排(D6/E1 Wave 8 co-seal)将新锁;沿用 firmware 行为可能与 D6 锁定后语义冲突
3. **codegen 等价非目标**:本工作项不追求生成 byte-equal 与 firmware FMS 当前 codegen 输出的 .c/.h(B4 仅要求 bus / enum / EXPORT struct 层面的契约 byte-equal,见 [B4 §… 退出条件](../B-contracts/B4-contract-diff.md));因此 leaf 实现允许重写。
4. **Phase 5+ 演进路径**:macro 层稳定后,leaf 层可独立重构(例如把 D4 shaper 公式从 "PX4-like jerk" 升级到 "MPC trajectory generation" 而不动 macro 拓扑)。
5. **OQ4 互动**:OQ4 选 transitional(§4.7),意味着 macro 层保留 `Control_Out_Bus` 输入对应的 6 子模块"接收口";leaf 层在 D2 设计时把该输入路由给 Safety + diagnostic 子模块(不创建新 setpoint 路径)— 此与 OQ2 macro 保留兼容。

#### 4.8.3 与 D2 / D3 / D4 / D5 / D6 的协调

D1 这条裁决产生以下下游约束:

- [D2 FMS 结构设计](D2-fms-structural.md):**必须**采用 6 子模块拓扑(macro 保留);子模块边界与内部信号自由设计(leaf 重写授权)
- [D3 Mode Manager](D3-mode-manager.md):Stateflow **重写**;状态名沿用 §4.1.1 mode 工作名,enum 数值锁 B2;转换规则参考 §4.1.4 类别矩阵但允许新设计(不必复制 firmware Stateflow 拓扑)
- [D4 Command Shaper](D4-command-shaper.md):整形公式 **重写**;允许采用比 firmware 当前更现代化的算法(例:S-curve trajectory)只要满足 §4.3 类别 + B3 PARAM
- [D5 多旋翼 leaf](D5-multicopter-leaf.md):多旋翼 mode-cmd 映射、起飞/降落 profile **重写**;参数名与 firmware `FMS_PARAM` 兼容性按 [B3 §4.4](../B-contracts/B3-parameter-schema.md) 锁定字段名
- [D6 FMS↔Controller 接口约定](D6-fms-controller-interface.md):cmd_mask 位语义 **重新锁定**(Wave 8 co-seal with E1);允许与 firmware 当前位含义 diverge,但必须显式 diff 记录在 D6 中

### 4.9 跨引用与下游耦合

D1 是 D 区起点,本工作项之后 D 区其余文件的功能输入均来自本文件:

| 下游 | D1 提供 |
|---|---|
| [D2](D2-fms-structural.md) | §4.1 mode set 反推 6 子模块边界;§4.6 输出字段类别 → Output Assembler 范围 |
| [D3](D3-mode-manager.md) (Wave 8 co-seal) | §4.1 mode 集 = Stateflow 状态列;§4.1.4 转换矩阵 = Stateflow 转换骨架;§4.1.3 mode-at-reset = init/reset 状态 |
| [D4](D4-command-shaper.md) (Wave 8 co-seal) | §4.3 整形策略类别 + §4.3.4 takeoff/landing profile = D4 输入需求 |
| [D6](D6-fms-controller-interface.md) (Wave 8 co-seal) | §4.6.2 cmd_mask 字段产出说明(D1 命名工作位,D6 锁语义) |
| [F3](../F-ins-contract/F3-consumption-rules.md) | §4.4 INS 触发条件 + §4.2.3 staleness 策略 → INS 字段消费侧矩阵的 FMS 部分(per [01-design-relationships §4.6](../01-design-relationships.md): `B5, D1, E1 ⇢ F3`) |
| [E1 Controller 功能](../E-controller/E1-controller-functional.md) (Wave 8 co-seal) | §4.1 `ctrl_mode` echo → E1 控制律分支选择;§4.6 setpoint 字段集 = E1 消费输入 |
| [H1 验证场景目录](../H-verification/H1-scenario-catalog.md) | §4.1 mode 集 + §4.4 failsafe 集 = H1 golden path + edge case 场景候选 |

## 5. 已知风险与悬而未决问题

- **TR-05 / TR-06 Battery 触发的"测量字段来源"未定**
  - 影响:`FMS_PARAM.18 / .19` 已锁阈值,但电池电压字段在哪个 bus 未确定 — 既不在 `INS_Out_Bus`(per A3 §4.4.4),也不在 `Pilot_Cmd_Bus` / `GCS_Cmd_Bus` / `Auto_Cmd_Bus`。可能由 H4 故障目录决定通过 stub 注入(MIL)+ firmware battery monitor topic(SIH)
  - 处置:open;由 [H4 故障注入目录](../H-verification/H4-fault-catalog.md) 协同决定接口;Phase 2 闭环切片若不含电量场景,可暂搁 placeholder
- **`MODE_ACRO`(B2 §4.4.5)与 `PilotMode` 缺成员的不对称**
  - 影响:B2 §4.4.4 `PilotMode` 9 成员中无 `PMODE_ACRO`;§4.4.5 `FlightMode` 含 `MODE_ACRO`;意味着 ACRO 不能由 pilot mode_switch 直接触发(只能由 GCS `CMDTYPE_MODE_CHANGE` 携带 `MODE_ACRO`)。Phase 2 多旋翼若不计划 ACRO 场景,M-12 行可暂为 placeholder
  - 处置:open;由 D5 多旋翼 leaf 决定 Phase 2 是否启用 M-12;若启用,需 GCS 路径补完
- **OQ4 transitional 退出的 firmware 真实依赖未验证**
  - 影响:§4.7.3 假设"firmware 侧不依赖 `Control_Out_Bus` 回送 FMS"。当前作业环境未挂载 firmware,无法直接 grep 验证。Phase 3 SIH 启动后必须显式确认
  - 处置:open;由 Phase 3 SIH 设计阶段(架构 v1 §15.3)的 reset/init sequencing 检查项捎带验证;若发现 firmware 真依赖该字段,本文件 §8 变更日志 + INDEX 决策日志同步,OQ4 改回 "永久启用"(A3 §4.10 选项 1)
- **OQ2 leaf 重写与 firmware 行为兼容的具体偏差未列**
  - 影响:§4.8.1 声明 "leaf 重写 + macro 保留",但具体 leaf 实现哪些行为与 firmware FMS 不一致(例如:`PMODE_OFFBOARD` 的 entry guard、TR-08 mode preconditions 拒绝时的 error_code 时机) — 这些差异要等 D3 / D4 / D5 完成后才能枚举
  - 处置:open;由 D3 / D4 / D5 在自身退出条件中各自登记 "与 firmware 当前 FMS 行为差异表",汇总到 INDEX 决策日志(D 区 batch 评审收尾时)
- **`MissionState` 是否进入 `FMS_Out_Bus` 未定**
  - 影响:per B2 §4.4.8 备注 "由 B1 锁定其存在性";若不在 bus 中,FMS 内部 enum,无法被 Controller / GCS / log 直接消费
  - 处置:open;由 B1 byte-level 表 + B4 首跑确认;若 B1 锁定 `mission_state` 字段存在,本文件 §4.6.3 行随之激活
- **TR-08 Mode preconditions invalid 的"拒绝转换"语义与 reset 协议交互**
  - 影响:§4.4.1 行 TR-08 设置 `error_code=ERR_MODE_INVALID` 但 mode 不变;若 pilot 反复请求被拒绝的 mode,error_code 持续置位 — 应在何时清除未明确(下一帧 / mode 真正切换时 / 一次性事件)
  - 处置:由 D3 状态机锁定清除规则
- **`Auto_Cmd_Bus.cmd_mask_request` 在 M-11 OFFBOARD 中的"采纳/拒绝"裁决标准**
  - 影响:§4.2.2 第 5 优先级仅给"FMS 决定是否采纳"原则;具体规则(允许哪些 bit 被 Auto 请求 / 哪些 FMS 强制覆盖)留 D6/E1 Wave 8
  - 处置:延后到 D6/E1 co-seal batch;D1 不锁

## 6. 退出条件复核

对照 [`00-design-plan.md §4.D`](../00-design-plan.md) D1 退出条件:

> **D1 退出条件原文**:mode 集合、命令源、整形策略、安全触发条件、mission 范围(仅闭环切片需要的子集);记录架构 v1 §17 OQ4(`Control_Out_Bus` 是否长期作为 FMS 输入)的决策;记录 OQ2(保留现有 FMS 结构 vs 重新撰写)的决策

| # | 退出条件原文(分项) | 本文档依据 | 状态 |
|---|---|---|---|
| 1 | mode 集合 | §4.1(12 主 mode + 4 failsafe sub-mode;§4.1.1 表 + §4.1.2 数量统计 + §4.1.3 reset 行为 + §4.1.4 转换矩阵)| 满足 |
| 2 | 命令源 | §4.2(6 命令源清单 + §4.2.2 仲裁优先级 + §4.2.3 staleness/validity + §4.2.4 mode-source 喂养表)| 满足 |
| 3 | 整形策略 | §4.3(§4.3.1 stick 预处理 + §4.3.2 rate/jerk 限制 + §4.3.3 mode-conditional override + §4.3.4 takeoff/landing profile + §4.3.5 共享原语)| 满足 |
| 4 | 安全触发条件 | §4.4(11 触发 TR-01..TR-11 + §4.4.2 优先级 + §4.4.3 debounce + §4.4.4 可恢复性)| 满足 |
| 5 | mission 范围(闭环切片必需子集)| §4.5(§4.5.1 在范围 + §4.5.2 显式排除 + §4.5.3 子状态)| 满足 |
| 6 | 架构 v1 §17 OQ4 决策(`Control_Out_Bus` 是否长期作为 FMS 输入)| §4.7(transitional + §4.7.2 5 条理由 + §4.7.3 三阶段退出计划)| 满足 |
| 7 | OQ2 决策(保留现有 FMS 结构 vs 重新撰写)| §4.8(macro 保留 + leaf 重写 + §4.8.2 5 条理由 + §4.8.3 对 D2..D6 的协调约束)| 满足 |

## 7. 下游影响

按 [`01-design-relationships.md §4.4 / §4.6 / §4.8`](../01-design-relationships.md) D1 出边:

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [D2 FMS 结构设计](D2-fms-structural.md) | → 强前置 | §4.1 mode set + §4.6 输出字段 → 6 子模块边界与功能映射;§4.8 macro 保留约束(必须沿用 6 子模块拓扑)|
| [D3 Mode Manager](D3-mode-manager.md)(Wave 8 co-seal) | → 强前置(经 D2)| §4.1 mode 集合 = Stateflow 状态列;§4.1.4 转换矩阵 = 转换骨架;§4.4 失效触发 = Stateflow 失效分支;§4.1.3 mode-at-reset = init 状态 |
| [D4 Command Shaper](D4-command-shaper.md)(Wave 8 co-seal)| → 强前置(经 D2)| §4.3 整形策略类别 + §4.3.4 profile 类别 + B3 PARAM 引用 |
| [D5 多旋翼 leaf](D5-multicopter-leaf.md) | → 强前置(经 D3/D4)| §4.1 mode 表 + §4.3 shaper + §4.5 mission scope + §4.8 leaf 重写授权 |
| [D6 FMS↔Controller 接口](D6-fms-controller-interface.md)(Wave 8 co-seal)| ↔ 协同 | §4.6.2 cmd_mask 工作位字段产出(D1 命名,D6 锁语义);§4.1.1 mode → ctrl_mode echo |
| [E1 Controller 功能](../E-controller/E1-controller-functional.md)(Wave 8 co-seal)| ⇢ 信息输入 | §4.1 `ctrl_mode` echo + §4.6.1 setpoint 字段集 → E1 控制律分支与环路启用 |
| [F3 INS 消费规则](../F-ins-contract/F3-consumption-rules.md) | ⇢ 信息输入 | §4.4 INS 触发条件 + §4.2.3 staleness 策略 → F3 INS 字段消费矩阵的 FMS 部分 |
| [H1 验证场景目录](../H-verification/H1-scenario-catalog.md) | ⇢ 信息输入 | §4.1 mode 集 + §4.4 failsafe 集 + §4.5 mission scope = H1 golden path / edge case 候选 |

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-08 | fms-architect (D1 author subagent) | 初稿;闭合 D1 全部 7 条退出条件;OQ4 选 transitional + 三阶段退出计划;OQ2 选 macro 保留 + leaf 重写 |

## Self-check

- [x] frontmatter 完整,字段值合法
- [x] 上游文档全部存在且 status ≥ reviewed(架构 v1、A1..A8、B1..B3 均已 reviewed,见任务 brief 上游 snapshot)
- [x] 退出条件逐条复核完成,每条均给出依据(§6 表 7 行覆盖 mode 集 / 命令源 / 整形 / 安全 / mission / OQ4 / OQ2)
- [x] 引用路径全部可点击访问(同 design 目录与架构 v1)
- [x] 不存在 RULES §5 禁则中的内容(无 .slx 截图、无 .m 代码、无 firmware 实现细节复述、无重复 bus/enum/param 字段表)
- [N/A] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记 — **本文件 contract_impact=no(功能设计;契约由 B 区拥有),N/A**
- [N/A] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径 — **本文件不镜像 firmware 契约,仅引用 B 区已镜像的字段名,N/A**(§3 依赖中保留 firmware @ `<pending hash>` 占位以备后续 OQ4 验证使用)
- [x] 下游影响已沿关系图识别完毕(§7 表覆盖 D2 / D3 / D4 / D5 / D6 / E1 / F3 / H1)
- [x] 文档不超出本工作项范围:不做 D2 子模块分解、不做 D3 Stateflow 详细、不写 D4 整形公式、不写 D5 多旋翼数值、不锁 D6 cmd_mask 位语义、不重定义 B1/B2/B3 字段
