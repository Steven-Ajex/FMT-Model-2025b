---
work_item: D4
title: FMS Command Shaper 详细设计
upstream: [架构v1, A2, A3, A4, A5, A6, A7, A8, B1, B2, B3, D1, D2, D3, D6, E1]
contract_impact: no
status: draft
authored_at: 2026-05-08
last_reviewed_at:
reviewer_verdict: none
---

# D4 FMS Command Shaper 详细设计

## 1. 目的

设计 FMS Command Shaper(架构 v1 §12.1.5;[D2 §4.3.5](D2-fms-structural.md))子模块的内部拓扑、各 mode 下的 setpoint 路由 / 整形 / 限速策略,以及作为 `cmd_mask` **单一写者**(per [D2 §4.6.2](D2-fms-structural.md))在每个 mode 下应输出的 cmd_mask 位组合与对应 `FMS_Out_Bus` setpoint 字段值的配对契约,从而闭合 [00-design-plan.md §4.D D4](../00-design-plan.md) 的退出条件 "各 mode 下输出 cmd_mask + setpoint 映射;rate/jerk 限制策略"。

## 2. 范围

**在范围:**

- Command Shaper 内部子块拓扑(Mode-Conditional Router / Stick Conditioning / Profile Generator / Rate-Jerk Limiter / Output Assembler 5 个内部块)
- Phase 2 多旋翼 12 主 mode + 4 failsafe sub-mode(per [D1 §4.1](D1-fms-functional.md))**逐 mode** 的 setpoint 来源 → `FMS_Out_Bus` 字段映射 + cmd_mask 工作位置位
- 每个 setpoint 槽(pos / vel / acc / att / rate / yaw / yaw_rate / throttle)的 rate / jerk 限制策略类别(类别层 — 数值常量留 D5 多旋翼 leaf)
- Bumpless transfer(mode 切换瞬时的 setpoint 跳变抑制)策略
- TAKEOFF / LAND / RTL profile 生成(类别层算法描述,公式骨架而非数值曲线)
- Stick conditioning(死区 + expo + 单位换算)的整形拓扑(per [D1 §4.3.1](D1-fms-functional.md))
- cmd_mask emission 流水线(D4 作为 cmd_mask 单一写者,在 D6 §4.4 mutex 规则与 [D6 §4.5](D6-fms-controller-interface.md) INS validity 政策下的 emission 守卫)
- D4 的 init / reset 内部状态行为(per [A6](../A-architecture/A6-init-reset-contract.md))
- D4 → D5 / D6 / E1 的 hand-off 接口:数值参数留 D5;cmd_mask 位语义守 D6;setpoint 由 E1 消费

**不在范围(由其他工作项处理):**

- Mode Manager Stateflow 状态 / 转换 / 守卫 — 由 [D3 Mode Manager 详细设计](D3-mode-manager.md) 处理(co-seal Wave 8 sibling)
- cmd_mask **位语义**(置位时 Controller 做什么 / 优先级表 / 互斥规则) — 由 [D6 FMS↔Controller 接口约定](D6-fms-controller-interface.md) 处理(co-seal Wave 8 sibling)
- Controller 各环路启用 / 旁通 / cascade 拓扑 — 由 [E1 Controller 功能设计](../E-controller/E1-controller-functional.md) 处理(co-seal Wave 8 sibling)
- 多旋翼 mode-cmd 数值常量 / 起飞 / 降落 profile 数值参数(climb_rate / descent_rate / flare_alt 等) — 由 [D5 FMS 多旋翼 leaf 设计](D5-multicopter-leaf.md) 处理(Wave 9)
- Source Selector 仲裁(选哪一路源进 Shaper) — 由 [D2 §4.3.1](D2-fms-structural.md) 锁定子模块边界 + [D1 §4.2](D1-fms-functional.md) 锁定仲裁优先级
- Mission Manager wp 推进 / 接受半径判定 — 由 [D2 §4.3.4](D2-fms-structural.md) 锁定子模块;Shaper 仅消费 `FMS_Mission_to_Shape_Bus.wp_target_pos_ned_m`
- bus / enum / parameter 字段级 schema — 由 [B1](../B-contracts/B1-bus-inventory.md) / [B2](../B-contracts/B2-enum-inventory.md) / [B3](../B-contracts/B3-parameter-schema.md) 处理
- Output Assembler 字段装配 / 单一边界 bus packer — 由 [D2 §4.3.6 / §4.6](D2-fms-structural.md) 处理(D4 输出止于 `FMS_Shape_to_Asm_Bus`)
- Failsafe 触发逻辑 / 阈值 — 由 [D1 §4.4](D1-fms-functional.md) + [D2 §4.3.3](D2-fms-structural.md) 处理(D4 仅消费 `FMS_ModeMgr_to_Shape_Bus.active_mode` 落到失效子模式后的路由指令)

## 3. 依赖

> **co-seal batch 声明**:本文件属于 Wave 8 `(D3, D4, D6, E1)` co-seal batch(见 [`01-design-relationships.md` §5 entry 2](../01-design-relationships.md))。本文件与 D3 / D6 / E1 三个**同 batch 兄弟**互相引用,允许在 status=draft 时构成依赖闭环(per [RULES §6 self-check item 2](../RULES.md))。

| 上游 | 引用位置 | 用途 |
|---|---|---|
| [架构 v1 §12.1.5](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | reviewed | Command Shaper 子模块定义:converts selected intent into bounded setpoints;owns slew / rate / jerk limiting here, not in Controller |
| [架构 v1 §13](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | reviewed | FMS 单速率 20 ms |
| [A2 命名与单位约定](../A-architecture/A2-naming-conventions.md) | reviewed | 字段单位 / 坐标系 / 量化记法(`_m` / `_mps` / `_mps2` / `_rad` / `_radps` / NED / body / quat ZYX) |
| [A3 模块边界信号清单](../A-architecture/A3-module-boundaries.md) | reviewed | §4.4.6.1 setpoint 字段清单;§4.6 passthrough 双类(D4 处理 with-shape 三字段) |
| [A4 跨速率边界](../A-architecture/A4-rate-boundaries.md) | reviewed | RB-04 (INS→FMS 10→20 ms ZOH) / RB-05 (FMS→Ctrl 20→5 ms) / RB-06 (external→FMS) |
| [A5 变体策略](../A-architecture/A5-variant-strategy.md) | reviewed | Phase 2 = 多旋翼;D4 拓扑 macro 共享,D5 leaf 注入数值 |
| [A6 Init/Reset 状态契约 §4.4.2 / §4.5](../A-architecture/A6-init-reset-contract.md) | reviewed | shaper 内部状态 reset 类别;`FMS_Out_Bus` setpoint reset = ZERO;cmd_mask reset = HARDCODED 全 0 |
| [A7 时间约定](../A-architecture/A7-time-conventions.md) | reviewed | profile 计时基于 `FMS_step` 入参 timestamp;dt 来源 |
| [A8 共享库块 §4.2 / §4.3](../A-architecture/A8-shared-library-roster.md) | reviewed | `Rate_Limiter_3d` / `Jerk_Limiter_3d` / `Deadzone_With_Linear_Bridge` / `LowPass_Filter_1st` Layer A 原语;`FMS_Command_Shaper_Shell` Layer B 包装 |
| [B1 §4.7](../B-contracts/B1-bus-inventory.md) | reviewed | `FMS_Out_Bus` setpoint 字段 ledger(D4 引用,不重定义) |
| [B2 §4.4.5 / §4.4.6 / §4.4.7 / §4.4.14](../B-contracts/B2-enum-inventory.md) | reviewed | `FlightMode` / `CtrlMode` / `FailsafeState` / `cmd_mask` 位号 |
| [B3 §4.4.2](../B-contracts/B3-parameter-schema.md) | reviewed | `FMS_PARAM` 43 字段(D4 引用 .06..10 takeoff/landing,.27 mission_default_speed,.28..29 RTL,.33..41 shaper limits,.40..41 stick) |
| [D1 §4.1 / §4.2 / §4.3 / §4.4 / §4.6](D1-fms-functional.md) | reviewed | mode 集合 + 命令源 + 整形策略类别 + 失效子模式 + setpoint 字段产出说明 |
| [D2 §4.1 / §4.2 / §4.3.5 / §4.5 / §4.6 / §4.8](D2-fms-structural.md) | reviewed | Command Shaper 子模块边界 + 输入/输出 bus + hand-off 列头 + reset 行为 |
| [D3 Mode Manager 详细设计](D3-mode-manager.md) | **draft, co-seal batch (D3, D4, D6, E1)** | 提供 `FMS_ModeMgr_to_Shape_Bus`(active_mode + ctrl_mode_hint + 路由指令 + shaping enable 位);D4 §4.1 / §4.2 直接消费 |
| [D6 FMS↔Controller 接口](D6-fms-controller-interface.md) | **draft, co-seal batch (D3, D4, D6, E1)** | §4.2 cmd_mask 位语义;§4.3 优先级表;§4.4 互斥规则(D4 emission 守卫);§4.5 INS validity 政策;§4.6.2 D4 hand-off 列头;§4.7 reset / disarmed 强制 cmd_mask=0 |
| [E1 Controller 功能设计](../E-controller/E1-controller-functional.md) | **draft, co-seal batch (D3, D4, D6, E1)** | E1 消费 D4 产出的 setpoint 字段(per cmd_mask 位)→ E1 cascade 拓扑反向约束 D4 setpoint 的物理含义 |

**Firmware 引用(per [RULES §5](../RULES.md) 镜像政策)**:

- `FMT-Firmware/src/model/fms/<vehicle>/lib/FMS_types.h`(`FMS_Out_Bus` setpoint / cmd_mask / passthrough-with-shape 字段;commit `FMT-Firmware @ <pending hash>`)

> 注:`<pending hash>` 占位符与 [A3](../A-architecture/A3-module-boundaries.md) / [A6](../A-architecture/A6-init-reset-contract.md) / [B1](../B-contracts/B1-bus-inventory.md) / [B2](../B-contracts/B2-enum-inventory.md) / [B3](../B-contracts/B3-parameter-schema.md) / [D6](D6-fms-controller-interface.md) 同期补齐(FMT-Firmware 当前未挂载)。

## 4. 设计内容

### 4.1 Command Shaper 内部拓扑

per [架构 v1 §12.1.5](../../architecture/2026-05-05-fmt-model-architecture-v1.md) + [D2 §4.3.5 / §4.5](D2-fms-structural.md):Command Shaper 是**单一 subsystem**,内部 5 个块**全部 20 ms 周期**(per [架构 v1 §13](../../architecture/2026-05-05-fmt-model-architecture-v1.md);A4 RB-04/05/06 latch 在 Source Selector 与 Output Assembler 边界)。

#### 4.1.1 内部框图

```text
                     +---------------------------------- inputs (20 ms latched) ----------------------------------+
                     |                                                                                            |
   FMS_SrcSel_Bus ---+ raw setpoint sources:                                                                       |
                     |   (a) Pilot stick: roll/pitch/yaw/throttle stick_n01                                        |
                     |   (b) Auto target: target_pos / vel / acc / yaw / yaw_rate (M-11 OFFBOARD only)             |
                     |   (c) GCS setpoint update: target_pos / yaw (CMDTYPE_SETPOINT_UPDATE)                       |
   FMS_ModeMgr_to_Shape_Bus ----+                                                                                  |
                     | active_mode + ctrl_mode_hint + route_pos_from / route_vel_from / route_att_from /           |
                     | route_rate_from / route_throttle_from + enable_jerk_lim / enable_tilt_lim /                 |
                     | enable_takeoff_profile / enable_landing_profile / enable_rtl_profile                        |
   FMS_Mission_to_Shape_Bus ----+                                                                                  |
                     | wp_target_pos_ned_m + wp_target_yaw_rad + mission_phase (TAKEOFF/CRUISE/LAND/RTL_PHASE_*)   |
                     | mission_default_speed_mps echo                                                              |
   INS_Out_Bus (latched) ----+                                                                                     |
                     | position_ned_m / velocity_ned_mps / attitude_quat (used for: bumpless latch source +        |
                     | RTL 'face-home' yaw + ground / takeoff detection)                                           |
   FMS_PARAM ----+                                                                                                 |
                     | shaper limits (.33..41) + takeoff/landing/RTL profile params (.06..10, .27..29) +           |
                     | stick deadzone (.40) + manual hover throttle (.41)                                           |
                     |                                                                                            |
                     +-------+---------+----------+--------------------+----------+----------+-------------------+
                             |         |          |                    |          |          |
                             v         v          v                    v          v          v
                  +----------------------------------------------------------------------------+
                  | Block (1) Mode-Conditional Router                                          |
                  |   - reads active_mode + route_*_from indicators                            |
                  |   - selects one of {PILOT_STICK, AUTO_TARGET, MISSION_WP, HOLD_LATCH,      |
                  |     PROFILE, ZERO} as raw input for each setpoint slot                     |
                  |   - HOLD_LATCH source = mode-entry snapshot of natural setpoint            |
                  |     (for bumpless transfer; see §4.4)                                      |
                  |   - PROFILE source = output of Block (3) Profile Generator                 |
                  +-------+-------------+--------------+--------------+--------------+---------+
                          | pilot_stick |  auto_tgt    |  wp_tgt      |  hold_latch  |  zero
                          v             |              |              |              |
                  +----------------+    |              |              |              |
                  | Block (2)      |    |              |              |              |
                  | Stick          |    |              |              |              |
                  | Conditioning   |    |              |              |              |
                  |  - deadband    |    |              |              |              |
                  |    (PARAM .40) |    |              |              |              |
                  |  - expo curve  |    |              |              |              |
                  |    (D5 leaf)   |    |              |              |              |
                  |  - unit-map to |    |              |              |              |
                  |    rad / radps |    |              |              |              |
                  |    / mps / 0..1|    |              |              |              |
                  +----+-----------+    |              |              |              |
                       |                |              |              |              |
                       |                |              v              |              |
                       |                |     +------------------+    |              |
                       |                |     | Block (3)        |    |              |
                       |                |     | Profile Gen.     |    |              |
                       |                |     |  - TAKEOFF z(t)  |    |              |
                       |                |     |  - LAND vz(t)    |    |              |
                       |                |     |  - RTL seq       |    |              |
                       |                |     |    (climb→cruise |    |              |
                       |                |     |     →hold→land)  |    |              |
                       |                |     +--------+---------+    |              |
                       |                |              |              |              |
                       v                v              v              v              v
                  +-------------------------------------------------------------------+
                  | Block (4) Rate / Jerk Limiter (per slot)                          |
                  |   - vel: Rate_Limiter_3d  (PARAM .33 / .34 / .35)                 |
                  |   - acc: Rate_Limiter_3d (PARAM .36) + Jerk_Limiter_3d (PARAM .37)|
                  |   - att: tilt-limit clamp (PARAM .38)                             |
                  |   - rate: rate-saturate (D5 leaf)                                 |
                  |   - yaw_rate: rate-saturate (PARAM .39)                           |
                  |   - throttle: clamp 0..1 + low-pass (D5 leaf, optional)           |
                  +-------+-----------------------------------------------------------+
                          |
                          v
                  +-------------------------------------------------------------------+
                  | Block (5) Output Assembler (Shaper-internal — distinct from       |
                  |    D2 §4.3.6 (6) Output Assembler that builds FMS_Out_Bus)        |
                  |  - assembles FMS_Shape_to_Asm_Bus payload:                        |
                  |    * 8 setpoint fields (pos/vel/acc/att_quat/att_eul/rate/yaw/    |
                  |       yaw_rate/throttle)                                          |
                  |  - emits cmd_mask uint32 bitfield (8 bits Phase 2)                |
                  |     * applies D6 §4.4 mutex check before emit                     |
                  |     * applies D6 §4.5 INS validity gate                           |
                  |  - passes passthrough-with-shape fields:                          |
                  |     actuator_cmd[] / pilot_throttle_passthrough /                 |
                  |     pilot_yaw_rate_passthrough_radps                              |
                  +---------+---------------------------------------------------------+
                            |
                            v
                  FMS_Shape_to_Asm_Bus  (to D2 §4.3.6 (6) Output Assembler)
```

#### 4.1.2 5 个内部块的一句话职责

| # | 块 | 一句话职责 | A8 原语依赖 |
|---:|---|---|---|
| (1) | **Mode-Conditional Router** | 按 `FMS_ModeMgr_to_Shape_Bus` 的 `route_*_from` 指令把 6 类原始 setpoint 来源(PILOT_STICK / AUTO_TARGET / MISSION_WP / HOLD_LATCH / PROFILE / ZERO)路由到 8 个输出槽 | `Multiport_Switch`(基础 Simulink 块,无 A8 原语) |
| (2) | **Stick Conditioning** | pilot stick 的死区 + expo + 单位换算 + 经 mode-条件 max 限值映射(per [D1 §4.3.1](D1-fms-functional.md))| [`Deadzone_With_Linear_Bridge`](../A-architecture/A8-shared-library-roster.md) |
| (3) | **Profile Generator** | TAKEOFF / LAND / RTL 三个 mode-specific profile 的时间序列生成(类别层算法 — 数值常量 D5 leaf)| [`Linear_Trajectory_Generator`](../A-architecture/A8-shared-library-roster.md)(若 A8 已锁;否则块内 truth-table)+ 内部 step-counter |
| (4) | **Rate / Jerk Limiter** | 每槽独立的 rate / jerk 限制(per [D1 §4.3.2](D1-fms-functional.md));per-mode shaping_enable 位决定是否激活 | [`Rate_Limiter_3d`](../A-architecture/A8-shared-library-roster.md) / [`Jerk_Limiter_3d`](../A-architecture/A8-shared-library-roster.md) |
| (5) | **Output Assembler**(shaper-internal)| 装配 `FMS_Shape_to_Asm_Bus` 包(8 setpoint 字段 + cmd_mask + 3 passthrough-with-shape 字段);emission 前做 D6 §4.4 mutex 检查 + D6 §4.5 INS validity gate | bus packer + bit-OR / bit-AND assembly |

> **命名解歧**:Block (5) 是 Shaper 内部的"装配 + cmd_mask 写入"块;**与** [D2 §4.3.6](D2-fms-structural.md) 的 (6) Output Assembler(产出 `FMS_Out_Bus` 单一边界)是**两个不同子模块**。Shaper Block (5) 输出 `FMS_Shape_to_Asm_Bus` 给 D2 (6),后者再装配 `FMS_Out_Bus`。

#### 4.1.3 子模块单一性约束(per [D2 §4.3.5](D2-fms-structural.md))

- Shaper 是**单一 subsystem**,不再细分到顶层;5 个内部块全部位于 Shaper subsystem 内
- Shaper **无 Stateflow**;mode-conditional 路由用 multiport switch / variant subsystem;profile 计时用基础块 + step-counter
- Shaper **不持有** mode 状态;`active_mode` 由 D3 单点产生,通过 `FMS_ModeMgr_to_Shape_Bus` 进入(per [D2 §4.2.2](D2-fms-structural.md) 职责正交性约束 #4)

### 4.2 Per-mode setpoint 映射 + cmd_mask emission 表

D4 在每个 mode 下按下表逐槽路由 setpoint 来源 + 选择整形原语 + 设置 cmd_mask 工作位。表中:

- **来源**列符号:`PILOT` = Pilot stick(经 Block (2));`AUTO` = `Auto_Cmd_Bus.target_*`;`MISSION` = `wp_target_*`;`HOLD@entry` = mode-entry 快照(bumpless latch,§4.4);`PROFILE` = Block (3) 输出;`ZERO` = 字段值 = 0;`face-home` = 由 INS 当前 pos 与 home 计算的 yaw 目标;`HOME` = `FMS_Out_Bus.home_*` echo 派生
- **整形**列符号:`DZ` = deadzone(PARAM .40);`EXPO` = expo curve(D5 leaf);`UNIT-MAP` = stick_n01 → 物理单位(per [D1 §4.3.1](D1-fms-functional.md));`RATE-LIM` = Rate_Limiter_3d;`JERK-LIM` = Jerk_Limiter_3d;`TILT-LIM` = tilt 角饱和;`CLAMP` = scalar 限幅;`PROFILE-GEN` = Profile Generator output;`PASS` = pass-through(无整形);`HOLD` = latch 上一帧值(无整形)
- **cmd_mask 位**符号:`POS` = `MASK_BIT_POSITION_LOOP`;`VEL` = `MASK_BIT_VELOCITY_LOOP`;`ACC` = `MASK_BIT_ACCELERATION_LOOP`;`ATT` = `MASK_BIT_ATTITUDE_LOOP`;`RATE` = `MASK_BIT_RATE_LOOP`;`YAW` = `MASK_BIT_YAW_LOOP`;`YAWR` = `MASK_BIT_YAW_RATE_LOOP`;`THR` = `MASK_BIT_THROTTLE_PASSTHROUGH`(位号、底层整型由 [B2 §4.4.14](../B-contracts/B2-enum-inventory.md) 锁;语义由 [D6 §4.2](D6-fms-controller-interface.md) 锁)

#### 4.2.1 主 mode 表(M-01 .. M-12,per [D1 §4.1.1](D1-fms-functional.md))

| Mode | pos_cmd_ned_m | vel_cmd_ned_mps | acc_cmd_ned_mps2 | att_cmd_quat | ang_rate_cmd_b_radps | yaw_cmd_rad | yaw_rate_cmd_radps | throttle_cmd | cmd_mask 位组合 | 备注 |
|---|---|---|---|---|---|---|---|---|---|---|
| **M-01 IDLE / DISARMED** | ZERO | ZERO | ZERO | identity | ZERO | 0 | 0 | 0 | `0` (全清) | 强制态;per [A6 §4.4.2](../A-architecture/A6-init-reset-contract.md) + [D6 §4.7](D6-fms-controller-interface.md) |
| **M-02 MANUAL** | ZERO | ZERO | ZERO | identity | PILOT × DZ + EXPO + UNIT-MAP(roll/pitch/yaw stick → body rate)+ RATE-LIM | 0 | PILOT yaw stick × DZ + EXPO + UNIT-MAP + CLAMP(.39) | PILOT throttle stick × PASS(0..1)| `RATE+YAWR+THR` `(0,0,0,0,1,0,1,1)` | per [D6 §4.4.1](D6-fms-controller-interface.md) 合法核心 |
| **M-03 STABILIZE** | ZERO | ZERO | ZERO | PILOT × DZ + UNIT-MAP(roll/pitch stick → tilt 角)+ TILT-LIM(.38);yaw 分量由 yaw_rate 积分(see col.7 / col.8)| ZERO | PILOT yaw stick → 积分到 yaw 角(or 0 if HOLD@entry)| PILOT yaw stick × DZ + EXPO + CLAMP(.39) | PILOT throttle stick × PASS | `ATT+YAWR+THR` `(0,0,0,1,0,0,1,1)` | tilt 由 attitude 路径,yaw 由 rate 路径(yaw 通道三态 per [D6 §4.3.5](D6-fms-controller-interface.md))|
| **M-04 ALTHOLD** | ZERO | `[0, 0, PILOT throttle stick × DZ + UNIT-MAP + CLAMP(.34/.35)]` | ZERO | PILOT × DZ + UNIT-MAP(roll/pitch → tilt)+ TILT-LIM(.38)| ZERO | 0 | PILOT yaw stick × DZ + EXPO + CLAMP(.39) | 0(闭环)| `VEL+ATT+YAWR` `(0,1,0,1,0,0,1,0)` | vel 仅 z 轴;水平由 ATT cascading;throttle 闭环不直传 |
| **M-05 POSHOLD** | HOLD@entry(水平)+ stick offset 积分 / `HOLD` 当 stick==0;垂直由 vel 积分推导但实际 D4 走 vel 路径(see col.3) | `[PILOT stick × DZ + UNIT-MAP + RATE-LIM(.33), 同, PILOT throttle stick → vz × CLAMP(.34/.35)]` | JERK-LIM(.37,水平 2 轴);ZERO 垂直 | identity(由 thrust-vector 推导,per [D6 §4.3.6](D6-fms-controller-interface.md);D4 不填)| ZERO | HOLD@entry / pilot yaw stick → 积分(yaw intervention,per [D1 §4.3.3](D1-fms-functional.md))| PILOT yaw stick × DZ + EXPO + CLAMP(.39)(yaw FF)| 0(闭环)| `POS+VEL+ACC+YAW+YAWR` `(1,1,1,0,0,1,1,0)` | jerk-limited 速度跟随 stick;pos 与 vel 同时启用为典型 trajectory FF |
| **M-06 TAKEOFF** | `[HOME.x, HOME.y, HOME.z - PROFILE-GEN.takeoff_alt(t)]`(由 PARAM .06 终值 + .07 climb rate)| `[0, 0, PROFILE-GEN.vz(t)]`(= `-takeoff_climb_speed_mps` 期间;到达后 ZERO)| ZERO | identity | ZERO | HOLD@entry(face-launch yaw)| 0 | 0(闭环)| `POS+VEL+YAW` `(1,1,0,0,0,1,0,0)` | profile 由 Block (3) 生成;到达 takeoff_alt 后 D3 切 M-05 POSHOLD |
| **M-07 LAND** | HOLD@entry(水平,LAND 不动 horizontal pos)| `[0, 0, PROFILE-GEN.vz(t)]`(`landing_descent_speed_mps` .08,低于 flare_alt 切 .09)| ZERO | identity | ZERO | HOLD@entry | 0 | 0(闭环)| `POS+VEL+YAW` `(1,1,0,0,0,1,0,0)` | flare_alt + 触地检测留 D5 leaf;触地后 D3 启动 .10 disarm dwell |
| **M-08 RTL** | PROFILE-GEN.rtl_pos(t)(分阶段:current → climb → home.xy → home + .28 → land)| RATE-LIM(.33 水平,.34/.35 垂直) + JERK-LIM(.37) | JERK-LIM | identity | ZERO | face-home(由 INS pos + home pos 计算 atan2)→ profile 末段切 HOLD | 0 | 0(闭环)| `POS+VEL+ACC+YAW` `(1,1,1,0,0,1,0,0)` | RTL 序列 4 段(climb / cruise / loiter / land)由 Block (3) 内部 phase counter 维护 |
| **M-09 LOITER**(航点)| `wp_target_pos_ned_m`(MISSION,等于当前 wp;停留期间 HOLD)| ZERO | ZERO | identity | ZERO | wp_target_yaw_rad if specified else HOLD@entry | 0 | 0(闭环)| `POS+YAW` `(1,0,0,0,0,1,0,0)` | per [D6 §4.4.1](D6-fms-controller-interface.md) "M-05 LOITER / M-08 hold" 等价合法行 |
| **M-10 MISSION** | MISSION wp_target(当前→下一 wp 直线段插值)| MISSION direction × `mission_default_speed_mps`(.27)+ RATE-LIM | JERK-LIM(.37) | identity | ZERO | wp_target_yaw_rad / 飞行方向 atan2 | 0 | 0(闭环)| `POS+VEL+ACC+YAW` `(1,1,1,0,0,1,0,0)` | mission trajectory 二阶 FF(D5 可启用三阶,与 ACC 一致) |
| **M-11 OFFBOARD** | AUTO target_pos_ned_m(若 Auto cmd_mask_request 含 POS) | AUTO target_vel_ned_mps × RATE-LIM(.33/.34/.35) | AUTO target_acc_ned_mps2 × CLAMP(.36) + JERK-LIM(.37) | AUTO target_quat × TILT-LIM(.38) | AUTO target_ang_rate(若 Auto cmd_mask_request 含 RATE)| AUTO target_yaw_rad | AUTO target_yaw_rate × CLAMP(.39) | AUTO throttle(若含 THR)| **由 D3 仲裁 Auto 的 cmd_mask_request 后**经 §4.7 mutex / validity gate 输出 | per [D6 §4.6.1 + §5 risk "Auto_Cmd_Bus.cmd_mask_request 采纳标准"](D6-fms-controller-interface.md) — D4 接受 D3 已经过滤的最终位组合 |
| **M-12 ACRO** | ZERO | ZERO | ZERO | identity | PILOT × DZ + EXPO + UNIT-MAP(stick → body rate)+ CLAMP(D5 leaf rate-max) | 0 | PILOT yaw stick × DZ + EXPO + CLAMP(.39) | PILOT throttle stick × PASS | `RATE+YAWR+THR` `(0,0,0,0,1,0,1,1)` | 同 M-02 但 stick 映射到更高 rate 上限(D5 lock)|

#### 4.2.2 Failsafe sub-mode 表(FS-01 .. FS-04,per [D1 §4.1.1](D1-fms-functional.md) failsafe-derived 表)

Failsafe sub-mode 是通过修改 `mode` + `failsafe_state` 实现的 mode 降级,**不是独立 mode**。D4 在收到 `FMS_ModeMgr_to_Shape_Bus.active_mode` 已被切到对应 mode 时按下表行为:

| Sub-mode | 等价 mode | D4 行为 | cmd_mask 位组合 | 备注 |
|---|---|---|---|---|
| **FS-01 RTL(failsafe)** | M-08 RTL | **同 M-08**;Profile Generator 强制启用 RTL profile;路由指令由 D3 推送(`enable_rtl_profile=1`) | 同 M-08:`POS+VEL+ACC+YAW` `(1,1,1,0,0,1,0,0)` | failsafe_state 字段由 Output Assembler 装配;D4 setpoint 不需感知 |
| **FS-02 LAND-NOW** | M-07 LAND | **同 M-07**;若当前 INS pos 不可信,降级为 vz-only(see FS-04 行为) | 同 M-07:`POS+VEL+YAW` `(1,1,0,0,0,1,0,0)` | critical 电量;D4 仍走 LAND profile |
| **FS-03 HOVER-HOLD** | M-05 POSHOLD | **同 M-05**;pilot stick 仍可作 vel offset(per [D1 §4.4 RTL → POSHOLD on FA_HOLD](D1-fms-functional.md))| 同 M-05:`POS+VEL+ACC+YAW+YAWR` `(1,1,1,0,0,1,1,0)` | GCS link loss + `failsafe_gcs_loss_action=FA_HOLD` |
| **FS-04 DISARM(forced)** | M-01 DISARMED | **强制全清** — 无论原 mode 路由指令;D4 输出 cmd_mask=`0`,所有字段 ZERO/identity;internal latches 按 D3 触发的 local reset 全清(§4.8) | `0`(全清)| INS 失效 / pilot kill_switch / motor fault;per [D6 §4.5 INS-完全失效行 + §4.7 reset 强制态](D6-fms-controller-interface.md) |

#### 4.2.3 mode 集合数量复核

- 主 mode:**12 项**(M-01 .. M-12;per [D1 §4.1.2](D1-fms-functional.md))→ §4.2.1 表 12 行
- Failsafe sub-mode:**4 项**(FS-01 .. FS-04)→ §4.2.2 表 4 行
- **总覆盖**:16 个 mode/sub-mode,与任务 brief 要求 "12 main modes + 4 failsafe sub-modes" 一致

### 4.3 Rate / Jerk 限制策略

D4 把 rate / jerk 限制集中在 Block (4) Rate-Jerk Limiter,**每槽独立**;**类别层** PARAM 引用如下表;**数值常量**留 [D5 多旋翼 leaf](D5-multicopter-leaf.md)。

#### 4.3.1 Per-slot 策略表

| Slot | 字段(per [A3 §4.4.6.1](../A-architecture/A3-module-boundaries.md) / [B1 §4.7](../B-contracts/B1-bus-inventory.md))| Rate limit class | Jerk limit class | Default PARAM(B3 .33..41)| Per-mode 覆盖 |
|---|---|---|---|---|---|
| **pos** | `pos_cmd_ned_m[3]` | **none**(pos 由 latch / profile / mission 离散输出,不经速率限制 — 速率限制在 vel 层落)| n/a | n/a | TAKEOFF / LAND / RTL:由 Profile Generator 内嵌 climb/descent 速率(.07 / .08 / .09 / .28),非外部 rate-limiter |
| **vel** | `vel_cmd_ned_mps[3]` | **per-axis hard saturation**(水平模值 ≤ `vel_xy_lim_mps`(.33);垂直 asymmetric `vel_z_up_lim_mps`(.34) / `vel_z_down_lim_mps`(.35))| **per-axis slew rate**(等价 acc 限,通过 Rate_Limiter_3d 输出 dV/dt → ≤ `acc_xy_lim_mps2`(.36))| .33 / .34 / .35 / .36 | AUTO modes (M-08 / M-10 / M-11):严格走完整 rate-lim;MANUAL / STABILIZE:vel 槽不启用(走 RATE 路径)|
| **acc** | `acc_cmd_ned_mps2[3]` | **hard saturation**(水平模值 ≤ `acc_xy_lim_mps2`(.36))| **second-order limit**(`jerk_xy_lim_mps3`(.37))| .36 / .37 | M-09 / M-10 / M-11 三阶 FF 启用 ACC + JERK-LIM;否则 ACC 槽 = ZERO |
| **att(quat)** | `att_cmd_quat[4]` | **tilt-angle saturation**(由 quat → tilt 计算后 clamp 至 `tilt_lim_rad`(.38) → 重构 quat,保持 yaw 分量不变)| n/a(姿态 jerk 由内层 RATE 限定)| .38 | M-11 OFFBOARD:Auto target_quat 经 .38 saturate;M-03 STABILIZE:stick 直映射后 saturate |
| **att(euler legacy)** | `att_cmd_euler_rad[3]` | 同 att(quat),先转 quat 再 saturate 再选输出 | n/a | .38 | E1 告警(per [D6 §4.3.4](D6-fms-controller-interface.md));legacy path |
| **rate** | `ang_rate_cmd_b_radps[3]` | **per-axis hard saturation**(rate-max,leaf-specific;**D5 锁定数值**)| **optional first-order LP**(`LowPass_Filter_1st`,per A8 §4.2;cutoff 由 D5 leaf 锁)| (D5 leaf field, **不在 .33..41 范围**;D5 fills in `D5-multicopter-leaf.md`)| MANUAL / ACRO 启用;闭环 mode RATE 槽 = ZERO |
| **yaw** | `yaw_cmd_rad` | **rate-of-change limit**(slew rate cap = `yaw_rate_lim_radps`(.39),通过对 yaw 角的微分检测)| n/a | .39 | 所有非 ZERO yaw 路径(M-03..M-11)启用;yaw 由 yaw_rate stick 积分时同时受 yaw_rate 限 |
| **yaw_rate** | `yaw_rate_cmd_radps` | **scalar hard saturation**(`yaw_rate_lim_radps`(.39))| n/a(可选 LP filter,D5 leaf)| .39 | MANUAL / STABILIZE / ACRO / 大部分 mode 的 yaw intervention 通道启用 |
| **throttle** | `throttle_cmd` | **scalar clamp** `0..1` | n/a(可选 first-order LP for stick smoothing,D5 leaf)| (无独立 PARAM;`manual_thr_hover_n01`(.41)是 stick 中位映射,非 limit)| MANUAL / STABILIZE / ACRO 直传时启用;闭环 mode = 0 |

> **关键不变量**:Rate 限以 `dt` = `FMS_step` 周期(20 ms,per [架构 v1 §13](../../architecture/2026-05-05-fmt-model-architecture-v1.md))为 dt 来源(per [A7 §4.6](../A-architecture/A7-time-conventions.md));若 `FMS_step` 入参 timestamp 跳变(非单调),rate-limiter 内部状态由 D3 触发的 local reset 清零(§4.8)。

#### 4.3.2 类别说明

- **hard saturation**:输出 = max(min(input, +lim), -lim);单点饱和,不依赖前帧
- **per-axis slew rate**:输出由前帧值通过 `output(t) = output(t-1) + clamp(input - output(t-1), -rate*dt, +rate*dt)` 推进;典型 [`Rate_Limiter_3d`](../A-architecture/A8-shared-library-roster.md) 实现
- **second-order limit (jerk)**:由 [`Jerk_Limiter_3d`](../A-architecture/A8-shared-library-roster.md) 完成 — 输入加速度命令经过 jerk-bounded 滤波器,产出 d(acc)/dt ≤ jerk_lim 的输出
- **tilt-angle saturation**:从 `att_cmd_quat` 提取 tilt 角(z-axis 偏离 NED-Down 的角度),若 > `tilt_lim_rad`,沿 tilt 轴等比例缩回 → 保留 quat yaw 分量 → 重构 quat
- **scalar clamp**:scalar 单点饱和

#### 4.3.3 Per-mode rate / jerk 启用矩阵

D3 通过 `FMS_ModeMgr_to_Shape_Bus.enable_*` 位告诉 Shaper 哪些限制启用:

| Mode | enable_rate_lim_vel | enable_jerk_lim | enable_tilt_lim | enable_yaw_rate_lim | enable_takeoff_profile | enable_landing_profile | enable_rtl_profile |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| M-01 IDLE | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| M-02 MANUAL | 0 | 0 | 0 | 1 | 0 | 0 | 0 |
| M-03 STABILIZE | 0 | 0 | 1 | 1 | 0 | 0 | 0 |
| M-04 ALTHOLD | 1 (vz only) | 0 | 1 | 1 | 0 | 0 | 0 |
| M-05 POSHOLD | 1 | 1 | 1 | 1 | 0 | 0 | 0 |
| M-06 TAKEOFF | 1 (vz only by profile)| 0 | 1 | 1 | **1** | 0 | 0 |
| M-07 LAND | 1 (vz by profile) | 0 | 1 | 1 | 0 | **1** | 0 |
| M-08 RTL | 1 | 1 | 1 | 1 | 0 | 1 (final phase) | **1** |
| M-09 LOITER | 0 (pos hold)| 0 | 1 | 1 | 0 | 0 | 0 |
| M-10 MISSION | 1 | 1 | 1 | 1 | 0 | 0 | 0 |
| M-11 OFFBOARD | 1 (per Auto request)| 1 (per Auto request)| 1 | 1 | 0 | 0 | 0 |
| M-12 ACRO | 0 | 0 | 0 | 1 | 0 | 0 | 0 |
| FS-01..FS-03 | 同等价 mode | 同 | 同 | 同 | 0 | 同 | 同 |
| FS-04 DISARM | 0 | 0 | 0 | 0 | 0 | 0 | 0 |

> **注**:enable_* 位由 D3 在 Stateflow 状态 `during` 上 emit;D4 直接消费,不再二次判 mode(per [D2 §4.2.2](D2-fms-structural.md) 职责正交性 #4)。

### 4.4 Bumpless transfer 策略

mode 切换时,新激活的 cmd_mask 位可能首次启用之前清位的 setpoint 槽 — 若该槽**首帧**取值与自然延续值差异过大,会经 Controller 转化为推力突变。Bumpless 策略由 D4 集中处理。

#### 4.4.1 Latching 政策

| 槽 | mode 切入时的"自然延续值"= latch 源 | 何时 latch | 何时清 |
|---|---|---|---|
| `pos_cmd_ned_m` | `INS_Out_Bus.position_ned_m` 当前帧值 | mode 切入第一帧(D3 emit `mode_changed=1` 单帧脉冲)| mode 退出时清(下次 latch 重新采样)|
| `vel_cmd_ned_mps` | `INS_Out_Bus.velocity_ned_mps` | 同 | 同 |
| `att_cmd_quat` | `INS_Out_Bus.attitude_quat` | 同 | 同 |
| `yaw_cmd_rad` | `INS_Out_Bus.attitude_quat` 提取 yaw | 同 | 同 |
| `vel_cmd_ned_mps[2]`(z 单独 — POSHOLD vz=0)| 0(POSHOLD 进入即静止)| mode 切入 | 同 |
| `throttle_cmd`(MANUAL → 闭环切换)| 上一帧 `throttle_cmd` 实际输出值 | mode 退出 MANUAL 时(切入闭环 mode 前)| 闭环 mode 期间不消费,仅供 reverse-direction switch 备用 |

#### 4.4.2 Soft-start ramp(可选)

- 在 latch 源与目标命令值差异 > 某阈值(D5 leaf-tunable)时,Block (4) Rate-Jerk Limiter 自动以 `ramp_t = transition_ramp_ms` 时间常数过渡(默认 200 ms,**D5 leaf 锁数值**)
- 实现:rate-limiter 的初始状态 = latch 源值;命令值 = mode-routed 目标;后续帧自然走 rate / jerk 限制路径
- **关键不变量**:bumpless ramp 不绕过 rate / jerk 限制;ramp 期间 rate-limiter 输出 ≤ PARAM .33..39 锁定的硬上限

#### 4.4.3 Discontinuous transitions(允许但 flagged)

某些 mode 转换(per [D1 §4.1.4](D1-fms-functional.md))天然跳变:

| 转换 | 不连续性 | D4 处置 |
|---|---|---|
| RTL → LAND(自动 RTL 末段) | RTL profile 阶段 4 的 vz 与 LAND profile 起步 vz 可能不同 | Block (3) Profile Generator 内部连接两个 profile,vz 由 RTL 末态 ramp 到 LAND .08 — 单一 generator 不出现 cmd_mask 位变化 |
| TAKEOFF → POSHOLD(到达高度后) | profile 输出 vz 从 .07 跳到 0 | Block (3) Profile Generator 末态主动输出 vz=0 至少 1 帧后再 D3 切 mode;**D4 保证最后一帧 vz=0**(详见 §4.5.1 末态)|
| MISSION → LOITER(wp arrived,wp 含 loiter) | vel 从 mission_default_speed 跳到 0 | Block (4) Rate-Jerk Limiter slew rate 限制 dV/dt;若 wp 接受半径与减速距离不匹配,允许过冲(D5 leaf 锁 wp_acceptance 与减速曲线对齐)|
| MANUAL → POSHOLD(pilot mode_switch) | rate stick 命令突然变成 pos hold(0 vel) | latch INS 当前 pos + vel 作为初值(§4.4.1);Block (4) ramp 速度 commanded → 0 经 .36 acc 限 |
| 任何 → DISARM(kill_switch / failsafe_INS) | cmd_mask 全 0 强制 | **不启用 bumpless ramp**;立即 ZERO + cmd_mask=0(per [D6 §4.7](D6-fms-controller-interface.md));Controller 收到 cmd_mask=0 在自身 local reset(per [A6 §4.5.2](../A-architecture/A6-init-reset-contract.md))中负责 motor 软停 |

#### 4.4.4 mode_changed 事件传递

D3 在 Stateflow 状态 `entry` 上 emit 单帧 `mode_changed=1` 脉冲(via `FMS_ModeMgr_to_Shape_Bus`);D4 Block (1) Mode-Conditional Router 收到该脉冲时:

1. 把所有 `route_*_from=HOLD_LATCH` 的槽的 latch 源采样进 latch register
2. 把 Block (4) Rate-Jerk Limiter 的 per-slot 内部状态(上一帧值)reset 到 latch 值(避免新 mode 首帧 rate-lim 起点错误)
3. Profile Generator(Block (3))内部 phase counter 复位到 phase_0(若启用了任何 profile)

### 4.5 Profile generation 详细

D4 在 Block (3) Profile Generator 内实现三个 mode-specific profile;**类别层算法**(time-driven 状态机 + 简单几何),**数值常量**留 [D5 leaf](D5-multicopter-leaf.md)。

#### 4.5.1 TAKEOFF profile

per [D1 §4.3.4 + B3 PARAM .06 / .07](D1-fms-functional.md):

```text
phase_0 (idle):                                     vz_out = 0;  pos_z_out = HOME.z
phase_1 (lift-off detect):                          vz_out = 0;  pos_z_out = HOME.z
   transition phase_0 → phase_1:
     when (active_mode == M-06 AND mode_changed pulse) → enter phase_1
phase_2 (climb):                                    vz_out = -takeoff_climb_speed_mps  (NED-Down: negative = up)
                                                    pos_z_out = HOME.z - integral(|vz_out| dt)
   transition phase_1 → phase_2:
     when (lift-off detected) → enter phase_2
     [lift-off detection = (pilot throttle stick > arm_throttle_thr_n01) AND (alt 微变测得 INS climb>threshold);
      具体阈值由 D5 leaf 锁]
phase_3 (terminal hold):                            vz_out = 0;  pos_z_out = HOME.z - takeoff_alt_m
   transition phase_2 → phase_3:
     when (HOME.z - INS.position_ned_m[2] >= takeoff_alt_m)  // PARAM .06
       → enter phase_3, emit profile_done=1 (1-frame pulse to D3)
       → D3 切 M-05 POSHOLD on profile_done
   末态保证: vz_out = 0 至少 1 帧后才允许 D3 切 mode (per §4.4.3)
```

#### 4.5.2 LAND profile

per [D1 §4.3.4 + B3 PARAM .08 / .09 / .10](D1-fms-functional.md):

```text
phase_0 (idle):                                     vz_out = 0
phase_1 (descent):                                  vz_out = +landing_descent_speed_mps  (NED-Down: positive = down)
   transition phase_0 → phase_1:
     when (active_mode == M-07 AND mode_changed pulse) → enter phase_1
phase_2 (flare — terminal slow descent):            vz_out = +land_touchdown_speed_mps  (PARAM .09)
   transition phase_1 → phase_2:
     when (-INS.position_ned_m[2] <= flare_alt_threshold)  // [D5 leaf-tunable, 默认 0.5..1 m above ground]
       → enter phase_2
phase_3 (touchdown detect + disarm dwell):          vz_out = 0; throttle = ramp to 0 over land_disarm_dwell_s
   transition phase_2 → phase_3:
     when (touchdown detected = INS.velocity 趋近 0 AND |INS.acc| spike OR contact sensor)
       // 触地检测算法 D5 leaf 锁
       → enter phase_3, emit profile_done after land_disarm_dwell_s (PARAM .10)
       → D3 切 M-01 DISARMED on profile_done
```

#### 4.5.3 RTL profile

per [D1 §4.3.4 + B3 PARAM .28 / .29 + LAND PARAMs](D1-fms-functional.md):

```text
phase_0 (idle):                                     pos_out = INS.position_ned_m (latched)
phase_1 (climb to RTL altitude):                    pos_out = [INS.x_at_entry, INS.y_at_entry, HOME.z - rtl_climb_alt_m]
                                                    vz_out = -takeoff_climb_speed_mps if below target alt else 0
   transition phase_0 → phase_1:
     when (active_mode == M-08 RTL AND mode_changed pulse) → enter phase_1
phase_2 (cruise to home, NED-horizontal):           pos_out = [HOME.x, HOME.y, HOME.z - rtl_climb_alt_m]
                                                    vel_out = direction_to_home × mission_default_speed_mps (.27 reuse)
   transition phase_1 → phase_2:
     when (|INS.position_ned_m[2] - target_alt| < 0.5 m)  // [D5 leaf]
phase_3 (loiter above home):                        pos_out = HOME (with rtl_climb_alt offset)
                                                    vel_out = 0
   transition phase_2 → phase_3:
     when (horizontal distance to HOME < wp_acceptance_radius_m (.26))
     duration = rtl_loiter_time_s (PARAM .29)
phase_4 (descend / land):                           hand-off to LAND profile (§4.5.2 phase_1)
                                                    yaw 切换为 face-home 不变
   transition phase_3 → phase_4:
     when (loiter_time elapsed >= .29) → enter phase_4
     → 进入 LAND phase_1 (vz_out = +.08)
   transition phase_4 (after LAND profile_done):
     → D3 切 M-01 DISARMED
```

> **yaw 在 RTL**:phase_1 latch 进入时的 yaw;phase_2 切到 face-home(由 INS pos + HOME pos 计算 atan2);phase_3 / phase_4 保持 face-home;具体过渡由 yaw 槽的 RATE-LIM(.39)控制(per §4.3.1 yaw 行)。

#### 4.5.4 MISSION profile(最简版)

per [D1 §4.5.1 + B3 PARAM .26 / .27](D1-fms-functional.md):

```text
Mission profile 由 D2 §4.3.4 (4) Mission/Auto Manager 提供 wp_target_pos_ned_m;
D4 Block (3) 仅做"current INS pos → wp_target 的直线 setpoint 插值",不持有 wp 数据库。

phase_traversal:                                    pos_out = wp_target_pos_ned_m  (mission 持续推送)
                                                    vel_out = direction_to_wp × mission_default_speed_mps (.27)
                                                    acc_out = jerk-limited from vel_out
   transition: 由 (4) Mission Manager 的 wp_reached 判定推送下一 wp
              D4 在 wp 切换帧由 (4) 内部自动新 wp_target_pos,Block (3) 自然连续
phase_loiter_at_wp (若 wp 含 loiter):              pos_out = wp_target_pos_ned_m
                                                    vel_out = 0
   transition: 由 D3 切到 M-09 LOITER (mode 切换,不在 D4 内部 phase counter)
```

#### 4.5.5 Profile Generator 内部状态

| 状态 | 类别 | reset 类别(per A6) |
|---|---|---|
| `takeoff_phase`(0..3 enum) | uint8 | HARDCODED 0 on init / local reset |
| `land_phase`(0..3 enum) | uint8 | HARDCODED 0 |
| `rtl_phase`(0..4 enum) | uint8 | HARDCODED 0 |
| `phase_entry_timestamp` | uint32 | HARDCODED 0;首次 phase 进入时由 `FMS_step` 入参 timestamp 写入(per A7) |
| `latched_entry_pos_ned_m[3]` | single[3] | ZERO on init / local reset;mode_changed 脉冲时由 INS 当前 pos 写入 |
| `profile_done`(1-frame pulse out) | uint8 | HARDCODED 0 |

### 4.6 Stick conditioning 详细

per [D1 §4.3.1 + B3 PARAM .40 / .41](D1-fms-functional.md);Block (2) Stick Conditioning 处理 4 个 pilot stick 通道(roll / pitch / yaw / throttle),共享一组整形原语。

#### 4.6.1 死区(Deadband)

per [D1 §4.3.1](D1-fms-functional.md) + [A8 §4.2 Deadzone_With_Linear_Bridge](../A-architecture/A8-shared-library-roster.md):

```text
def deadband(x, dz):
  if |x| < dz:                # PARAM .40 manual_stick_deadzone_n01,默认 0.05
    return 0
  else:
    # 线性桥接(避免 sign flip 处的阶跃)
    if x >= 0: return (x - dz) / (1 - dz)
    if x <  0: return (x + dz) / (1 - dz)
```

- 应用通道:roll / pitch / yaw / throttle stick
- PARAM:`manual_stick_deadzone_n01`(B3 .40)— 共享同一 deadzone(D5 可重写 per-axis,**留 D5**)

#### 4.6.2 Expo curve

per [D1 §4.3.1](D1-fms-functional.md):

```text
def expo(x, expo_ratio):
  # x ∈ [-1, +1] (deadband 后);expo_ratio ∈ [0, 1)
  return (1 - expo_ratio) * x + expo_ratio * x * x * x
```

- 应用通道:roll / pitch / yaw stick(throttle 通常不加 expo)
- PARAM:`expo_ratio_*`(D5 leaf-tunable;Phase 2 默认 0.0 = 线性 — D5 锁数值)
- D4 锁定**算法形式**(三次 polynomial blend);D5 锁系数

#### 4.6.3 单位换算(Unit map)

每通道按 mode-条件映射到物理单位;映射上限由 mode-specific PARAM 决定:

| Stick 通道 | Mode 类 | 物理目标单位 | 映射上限 PARAM |
|---|---|---|---|
| roll_stick_n01 | MANUAL / ACRO | body roll rate (rad/s) | (D5 leaf rate-max)|
| roll_stick_n01 | STABILIZE / ALTHOLD / 闭环 mode | tilt roll 角(rad)| `tilt_lim_rad`(.38) |
| roll_stick_n01 | POSHOLD | horizontal vel offset (m/s)| `vel_xy_lim_mps`(.33)|
| pitch_stick_n01 | 同 roll | 同 roll(对应 pitch 轴)| 同 |
| yaw_stick_n01 | 所有 mode | yaw rate(rad/s)| `yaw_rate_lim_radps`(.39)|
| throttle_stick_n01 | MANUAL / STABILIZE | throttle pass-through(0..1)| 无(直传 + clamp 0..1)|
| throttle_stick_n01 | ALTHOLD | vz cmd (m/s);中位 = 0,上推 = 上升 | `vel_z_up_lim_mps`(.34) / `vel_z_down_lim_mps`(.35) |
| throttle_stick_n01 | POSHOLD | 同 ALTHOLD | 同 |
| throttle_stick_n01 | AUTO modes (M-06..M-10) | **忽略**(per [D1 §4.3.3](D1-fms-functional.md))| n/a |

#### 4.6.4 Stick validity gate(per [A6 §4.5.3](../A-architecture/A6-init-reset-contract.md))

- 若 `Pilot_Cmd_Bus.valid==0`(staleness 触发或源未注入)→ Block (2) 输出 = 0(所有 stick 通道)
- 若 `Pilot_Cmd_Bus.kill_switch==1` → Block (2) 输出 = 0(等价 DISARM 信号)
- D4 不**直接**触发 mode 切换;失效经 [D2 §4.3.3](D2-fms-structural.md) Safety Monitor → Mode Manager 路径

### 4.7 cmd_mask emission

D4 是 cmd_mask 的**单一写者**(per [D2 §4.6.2](D2-fms-structural.md));emission 流水线在 Block (5) Output Assembler(shaper-internal)实现。

#### 4.7.1 emission 流水线

```text
Step 1: D3 推送 desired cmd_mask 模式 via FMS_ModeMgr_to_Shape_Bus.cmd_mask_request
        (per D2 §4.2.1 routing 指令的一部分;§4.2.1 没单独列出此字段名,
         D3 ↔ D4 在 Wave 8 sibling 内同步约定使用 cmd_mask_request 槽 — 候选名 D3 锁)

Step 2: D4 Block (5) 应用 D6 §4.4 mutex 检查
        - 若 cmd_mask_request 命中 MX-2 (THR & POS/VEL/ACC) → 强制清 BIT_THROTTLE
        - 若命中 MX-7 / MX-8 (无 thrust 源 / 无 attitude) → 强制全清,emit cmd_mask=0
        - 若命中 MX-9 (att_quat ≠ identity AND att_eul ≠ ZERO) → D4 必须保证只填一个字段
          (本 D4 设计保证:M-03 STABILIZE 等填 quat;legacy 通道未启用 — 
           euler 字段始终 ZERO),所以 MX-9 不会由 D4 触发

Step 3: D4 应用 D6 §4.5 INS validity gate
        从 INS_Out_Bus.INS_Status / INS_Flag 读取:
        - if INS_Status < INS_STATUS_READY: cmd_mask = 0 (forced, FS-04)
        - if INS_FLAG_BIT_POSITION_VALID == 0: clear BIT_POS
        - if INS_FLAG_BIT_VELOCITY_VALID == 0: clear BIT_VEL & BIT_ACC
        - if INS_FLAG_BIT_ATTITUDE_VALID == 0: cmd_mask = 0 (forced)
        - if INS_FLAG_BIT_HEADING_VALID == 0: clear BIT_YAW
        (D4 实现的此行为 = D6 §4.5 政策中"FMS 应 emit"的 emission 端 enforcement)

Step 4: D4 写 FMS_Shape_to_Asm_Bus.cmd_mask = 过滤后的 mask
        D4 同步把对应被清位的字段值 force ZERO/identity (per A6 §4.4.2 + D6 §4.6.2 契约
         "BIT_X = 1 ⇔ 字段值有意义;BIT_X = 0 ⇔ 字段值 = reset-time 默认")

Step 5: Output Assembler (D2 §4.3.6) 把 FMS_Shape_to_Asm_Bus.cmd_mask 直传写入
        FMS_Out_Bus.cmd_mask (Output Assembler 不修改位,per D2 §4.6.2)
```

#### 4.7.2 D4 实现的具体守卫(D6 mutex 子集)

由于 D3 已在 mode 表中只 emit [D6 §4.4.1 truth-table](D6-fms-controller-interface.md) 合法核心组合,D4 大部分情况下 cmd_mask_request 已合法;Block (5) 守卫主要拦截以下场景:

| 守卫 | 触发场景 | D4 行为 | 对应 D6 mutex 编号 |
|---|---|---|---|
| G-1 | Auto mode (M-11) 中 Auto request 含 THR + POS/VEL/ACC | 清 BIT_THROTTLE,保留 POS/VEL/ACC | MX-2 |
| G-2 | INS 完全失效 / `INS_Status < READY` | cmd_mask = 0;all setpoint = ZERO/identity | D6 §4.5 行 1 + MX-1 |
| G-3 | 部分 INS 失效(POS/VEL/ATT/HDG 单失效) | clear 对应位;同步 force 字段 ZERO | D6 §4.5 行 2..5 |
| G-4 | quat 与 euler 都填 | D4 设计保证只填一个;此守卫 redundant assert(debug-only)| MX-9 |
| G-5 | reset / DISARMED 强制态 | cmd_mask = 0 | MX-1 + D6 §4.7 |

> **关键不变量**:G-1..G-5 的实现**复制** D6 的政策(由 D6 §4.4 / §4.5 锁);D4 不**新增** mutex 规则。Phase 3+ 新增位 → D6 修订 → D4 修订(同步)。

#### 4.7.3 D4 不做的 cmd_mask 操作

- D4 **不**决定哪个 mode 在哪些位置位 — 这由 D3 通过 cmd_mask_request 推送
- D4 **不**修改 cmd_mask 经 Output Assembler(per [D2 §4.6.2](D2-fms-structural.md))
- D4 **不**响应 Controller 端 ErrorCode — Controller 端 mutex 检测见 [D6 §4.6.3](D6-fms-controller-interface.md);D4 端只做 emission 守卫(production-side gate)

### 4.8 Reset / init 行为

per [A6 §4.4.2 / §4.5](../A-architecture/A6-init-reset-contract.md) + [D2 §4.3.5 / §4.8.1 行 (5)](D2-fms-structural.md):

#### 4.8.1 FMS_init 时(per A6 §4.2 / §4.3)

| Shaper 内部状态 | reset 类别 | reset 后稳态值 |
|---|---|---|
| Block (1) Mode-Conditional Router 的 routing 输出选择 | HARDCODED | `route_*_from = ZERO`(全部槽路由到 ZERO 源)|
| Block (2) Stick Conditioning 上一帧值 | ZERO | 全 0(stick = 0 等价于 deadband 触发后输出 0)|
| Block (3) Profile Generator phase counter | HARDCODED | takeoff_phase / land_phase / rtl_phase = phase_0 |
| Block (3) `phase_entry_timestamp` | HARDCODED | 0 |
| Block (3) `latched_entry_pos_ned_m` | ZERO | (0, 0, 0) |
| Block (4) Rate-Jerk Limiter per-slot 上一帧值 | ZERO | 0 (per-slot)|
| Block (5) `cmd_mask` 输出 | HARDCODED | 0(全清)|
| Block (5) bumpless latch register(`pos_latch` / `vel_latch` / `att_latch` / `yaw_latch` / `throttle_latch`)| ZERO | 0 / identity / 0 |
| **shaped 输出字段** `pos_cmd_*` / `vel_cmd_*` / `acc_cmd_*` / `att_cmd_*` / `ang_rate_cmd_*` / `yaw_*` / `throttle_*` | per [A6 §4.4.2](../A-architecture/A6-init-reset-contract.md) | ZERO / identity(quat)|
| **passthrough-with-shape** `actuator_cmd[]` / `pilot_throttle_passthrough` / `pilot_yaw_rate_passthrough_radps` | ZERO | 0 |

#### 4.8.2 运行期 local reset(per [A6 §4.5.2](../A-architecture/A6-init-reset-contract.md))

由 D3 触发(via `FMS_Out_Bus.reset` 字段或 D3 chart action;per [A6 §4.5.2 表](../A-architecture/A6-init-reset-contract.md));D4 内部状态按 §4.8.1 表回到稳态值,**不**等待新 mode_changed 脉冲。

#### 4.8.3 mode_changed 触发的部分 reset(per §4.4.4)

mode_changed 不是 reset(per [A6 §4.5.2](../A-architecture/A6-init-reset-contract.md) 区分);是**latch 重新采样 + Rate-Jerk Limiter 内部状态重置到 latch 值**;不影响 cmd_mask emission(由 D3 决定)。

#### 4.8.4 DISARMED 期间(M-01)

- cmd_mask = 0(per [D6 §4.7](D6-fms-controller-interface.md))
- 所有 setpoint 字段 = ZERO / identity(per A6 §4.4.2)
- Profile Generator 全部 phase = phase_0(inactive)
- Block (4) Rate-Jerk Limiter 内部状态保持 ZERO(每帧重写)

### 4.9 Hand-off 表(给 D5 / D6 / E1)

per [01-design-relationships.md §4.4](../01-design-relationships.md) D 区出边 + [§4.5](../01-design-relationships.md) E 区出边;本节锁定 D4 → 下游的接口形状。

#### 4.9.1 D5 hand-off — 多旋翼数值常量

D5 在自己文档内填以下数值;D4 锁定**字段名**:

| D4 引用项 | D5 fills numerics |
|---|---|
| Stick deadband | 已锁:`manual_stick_deadzone_n01`(B3 .40)= 0.05 *(B4 verify)*;D5 复核多旋翼是否需偏离默认 |
| Stick expo coefficient | D5 锁:`expo_ratio_roll` / `_pitch` / `_yaw`(D5 leaf field;Phase 2 默认 0.0 线性)|
| Manual hover throttle | 已锁:`manual_thr_hover_n01`(B3 .41)= 0.5 *(B4 verify)* |
| TAKEOFF profile constants | 已锁:`takeoff_alt_m`(.06)= 2.5 m;`takeoff_climb_speed_mps`(.07)= 1.5 m/s;**lift-off detection threshold(D5 leaf)**;**phase_2 → phase_3 hysteresis(D5)** |
| LAND profile constants | 已锁:`landing_descent_speed_mps`(.08)= 0.7 m/s;`land_touchdown_speed_mps`(.09)= 0.3 m/s;`land_disarm_dwell_s`(.10)= 2.0 s;**flare_alt_threshold(D5 leaf)**;**触地检测算法 + 阈值(D5)** |
| RTL profile constants | 已锁:`rtl_climb_alt_m`(.28)= 30 m;`rtl_loiter_time_s`(.29)= 5 s;**rtl_climb_alt 容差(D5)**;**face-home yaw 转向 ramp(D5)** |
| Mission profile constants | 已锁:`wp_acceptance_radius_m`(.26)= 1.5 m;`mission_default_speed_mps`(.27)= 5 m/s;**减速曲线参数(D5)** |
| Shaper limits | 已锁:.33 / .34 / .35 / .36 / .37 / .38 / .39(B3,**MC-only**)|
| Rate-loop body-rate max(MANUAL / ACRO) | **D5 leaf field**(不在 .33..41 范围内)|
| Bumpless ramp duration | **D5 leaf field**(`transition_ramp_ms`,默认 200 ms;D5 复核)|
| LowPass filter cutoffs(stick smoothing / rate-loop)| **D5 leaf field** |

D5 兼容性 frame per [B3 §4.9](../B-contracts/B3-parameter-schema.md):D5 必须显式核对所有 `MC-only` 字段(per [B3 §4.4.2](../B-contracts/B3-parameter-schema.md))与 firmware `FMS_PARAM` 字段名兼容性。

#### 4.9.2 D6 hand-off — D4 honors D6

D4 §4.7.2 守卫表 G-1..G-5 是 [D6 §4.4 mutex](D6-fms-controller-interface.md) + [§4.5 INS gate](D6-fms-controller-interface.md) 的 production-side enforcement。具体对齐:

| D4 项 | D6 项 |
|---|---|
| §4.2.1 / §4.2.2 mode-cmd_mask 表 | [D6 §4.4.1 truth-table 合法行](D6-fms-controller-interface.md) |
| §4.7.2 G-1..G-5 守卫 | [D6 §4.4 MX-1..MX-10](D6-fms-controller-interface.md) + [§4.5 INS validity 表](D6-fms-controller-interface.md) |
| §4.8 reset / DISARMED cmd_mask=0 | [D6 §4.7 reset / disarmed 强制态](D6-fms-controller-interface.md) |
| §4.6.2 hand-off 列(setpoint 字段值契约)| [D6 §4.6.2](D6-fms-controller-interface.md) |

D6 反向约束 D4:任何 D4 输出位组合**必须**在 [D6 §4.4.1 合法核心](D6-fms-controller-interface.md) 中或属于 reset / disarmed 安全态。

#### 4.9.3 E1 hand-off — D4 produces, E1 consumes

D4 产出 8 个 setpoint 字段(per [D6 §4.6.2](D6-fms-controller-interface.md))+ cmd_mask;E1 按 [D6 §4.6.3](D6-fms-controller-interface.md) loop trim rules 解析。具体对齐:

| D4 输出 | E1 消费契约 |
|---|---|
| BIT_POS=1 ⇒ `pos_cmd_ned_m` 有效(NED 米)| E1 启用位置环;参考 = 该字段 |
| BIT_VEL=1 ⇒ `vel_cmd_ned_mps` 有效(NED m/s)| E1 启用速度环 |
| BIT_ACC=1 ⇒ `acc_cmd_ned_mps2` 有效;经 jerk-limited | E1 在速度环用作 FF |
| BIT_ATT=1 ⇒ `att_cmd_quat` 有效(identity-baseline);经 tilt-limit | E1 启用姿态环 |
| BIT_RATE=1 ⇒ `ang_rate_cmd_b_radps` 有效(body-frame rad/s)| E1 启用角速度环 |
| BIT_YAW=1 ⇒ `yaw_cmd_rad` 有效;经 yaw rate-of-change limit | E1 启用 yaw 角通道 |
| BIT_YAWR=1 ⇒ `yaw_rate_cmd_radps` 有效;经 .39 saturate | E1 启用 yaw 角速度通道 |
| BIT_THR=1 ⇒ `throttle_cmd` 有效(0..1);经 stick passthrough | E1 mixer 总推力直传 |
| 所有 BIT_X=0 ⇒ 对应字段 = ZERO/identity | E1 必须不消费(per [D6 §4.6.3](D6-fms-controller-interface.md)) |

E1 的 Mask Resolver(per [E2 §4.1.1](../E-controller/E2-controller-structural.md))在每帧入口解析此契约。

## 5. 已知风险与悬而未决问题

- **`cmd_mask_request` 字段在 `FMS_ModeMgr_to_Shape_Bus` 中的具体名未在 D2 §4.2.1 列出**
  - 影响:D4 §4.7.1 Step 1 描述 D3 通过 `cmd_mask_request` 槽传 desired mask;[D2 §4.2.1](D2-fms-structural.md) 行 `FMS_ModeMgr_to_Shape_Bus` 列了 routing 指令 + shaping enable 位,但未单独列 cmd_mask_request 字段。Wave 8 D3 author 应在自己文档锁此字段名;D4 文档保持引用占位
  - 处置:由 [D3 Mode Manager 详细设计](D3-mode-manager.md) Wave 8 sibling 在自己文档锁定字段名;若 D3 选用其他机制(如 routing 指令隐含 cmd_mask),D4 §4.7.1 Step 1 同步修订 §8 变更日志

- **MX-7 (THR-only,无 attitude) 在 Phase 2 MIL 不应出现,但 D4 emission 守卫未显式拦截**
  - 影响:[D6 §4.4 MX-7](D6-fms-controller-interface.md) 锁此组合"合法但极特殊;Phase 2 多旋翼 MIL 不应在飞行场景出现";D4 §4.7.2 G-1..G-5 没有"全平移 + ATT + RATE 全清 但 THR 置位"的拦截器(因为 Phase 2 D3 不 emit 此组合)
  - 处置:open;若 H1 / H4 引入 ground-test variant 触发 MX-7,D4 §4.7.2 增补 G-6 守卫(行为:emit warning + pass)。当前 Phase 2 D3 不 emit ⇒ D4 不需新增

- **Bumpless ramp duration `transition_ramp_ms` 默认值未锁,影响 mode 切换体感**
  - 影响:§4.4.2 ramp 实现依赖 D5 锁定数值;过长 → 模式切换迟钝;过短 → 突变。Phase 2 D5 决定数值
  - 处置:由 [D5 多旋翼 leaf 设计](D5-multicopter-leaf.md) Wave 9 锁定;D4 仅锁算法形式

- **Rate-loop body-rate max 不在 B3 .33..41 范围**
  - 影响:M-02 MANUAL / M-12 ACRO 需要 stick→body rate 映射上限,但 [B3 §4.4.2](../B-contracts/B3-parameter-schema.md) 字段表中没有 `rate_max_radps` 或类似字段。可能在 D5 leaf 自己定义 PARAM struct field,或经 [B3](../B-contracts/B3-parameter-schema.md) 增补
  - 处置:open;由 D5 author 与 B3 同步确认;若 B3 增补,B3 §10 changelog + D4 §3 引用同步

- **TAKEOFF lift-off 检测算法未锁数值阈值**
  - 影响:§4.5.1 phase_1 → phase_2 转换需"throttle stick > arm_throttle_thr_n01 AND alt 微变测得 INS climb > threshold";`arm_throttle_thr_n01` 已在 [B3 .02](../B-contracts/B3-parameter-schema.md) 锁;但"alt 微变 climb threshold"是 D5 leaf 字段
  - 处置:由 D5 leaf 锁

- **Profile Generator 与 Mission Manager 的"接力点"未明示在 D2 内部 bus 表**
  - 影响:RTL phase_4 → LAND profile 接力时,Mission Manager 不被卷入(LAND 不是 mission);但 RTL phase_2 cruise to home 用 `mission_default_speed_mps`(.27)— 这是 PARAM 复用,不需 Mission Manager
  - 处置:已闭合;Profile Generator 完全自包含,只读 PARAM 与 INS state

- **Stick conditioning 的 LowPass filter 是否在 Phase 2 启用未定**
  - 影响:§4.6.2 / §4.3.1 throttle row 提到"可选 first-order LP";启用与否 / cutoff 频率未锁
  - 处置:由 D5 leaf 锁;Phase 2 默认禁用(linear stick)

- **`Auto_Cmd_Bus.cmd_mask_request` 在 M-11 OFFBOARD 中的过滤规则**
  - 影响:[D6 §4.6.1 + §5 risk](D6-fms-controller-interface.md) 同 issue;Auto 给的 mask request 与 FMS 实际 emit 的差异规则未量化。D4 §4.2.1 行 M-11 仅说"由 D3 仲裁后采纳",D4 不另增过滤
  - 处置:由 [D3](D3-mode-manager.md) sibling 在自己文档锁过滤规则;D4 接受 D3 输出

## 6. 退出条件复核

对照 [`00-design-plan.md` §4.D 的 D4 行](../00-design-plan.md):退出条件 = "各 mode 下输出 cmd_mask + setpoint 映射;rate/jerk 限制策略"

| # | 退出条件原文 | 本文档依据 | 状态 |
|---|---|---|---|
| 1 | **各 mode 下输出 cmd_mask + setpoint 映射** | §4.2.1(12 主 mode 表,10 列 setpoint 字段映射 + cmd_mask 工作位组合)+ §4.2.2(4 failsafe sub-mode 表)+ §4.2.3 数量复核(16 mode/sub-mode 全覆盖)+ §4.7 cmd_mask emission 流水线 + §4.9.2/§4.9.3 D6/E1 hand-off | 满足 |
| 2 | **rate / jerk 限制策略** | §4.3.1 per-slot 9 行表(pos / vel / acc / att / rate / yaw / yaw_rate / throttle 各槽 rate / jerk class + PARAM 引用 + per-mode 覆盖)+ §4.3.2 类别说明 + §4.3.3 per-mode 启用矩阵(16 行)+ §4.4 bumpless transfer + §4.5 三个 profile 生成 | 满足 |

## 7. 下游影响

按 [`01-design-relationships.md §4.4`](../01-design-relationships.md) D 区出边 + [§4.5](../01-design-relationships.md) E 区出边:

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [D3 Mode Manager](D3-mode-manager.md)(Wave 8 sibling)| ↔ co-seal | §4.2.1 / §4.2.2 mode-cmd_mask 表锁定 D3 在每个状态 emit 的 cmd_mask 模式集合;§4.4.4 mode_changed 脉冲约定;§4.7.1 Step 1 cmd_mask_request 槽约定 |
| [D5 多旋翼 leaf](D5-multicopter-leaf.md)(Wave 9)| → 强前置 | §4.9.1 hand-off 表锁数值常量列(takeoff / landing / RTL profile / stick expo / rate-loop max / bumpless ramp / LP filter cutoffs)|
| [D6 FMS↔Controller 接口](D6-fms-controller-interface.md)(Wave 8 sibling)| ↔ co-seal | §4.7.2 G-1..G-5 守卫复制 D6 §4.4 mutex + §4.5 INS gate;§4.2 表合法行落 [D6 §4.4.1 truth-table](D6-fms-controller-interface.md) 中;§4.7 D4 是 cmd_mask 单一写者 enforcement |
| [E1 Controller 功能设计](../E-controller/E1-controller-functional.md)(Wave 8 sibling)| ⇢ 信息输入(经 D6) | §4.9.3 hand-off 表 — D4 setpoint 字段值契约 + cmd_mask 位 → E1 cascade 拓扑反向约束;§4.3 rate/jerk 限制将 setpoint 控制在物理可行范围,降低 E1 限幅压力 |
| [F3 INS 消费规则](../F-ins-contract/F3-consumption-rules.md)(Wave 9)| ⇢ 信息输入 | §4.7.1 Step 3 INS validity gate(production-side enforcement)+ §4.4.1 latching 政策(消费 INS 当前 pos / vel / quat 作为 latch 源)|
| [G3 日志与可观测性](../G-harness/G3-logging.md)(Wave 10)| ⇢ | §4.5 三个 profile 内部状态(takeoff_phase / land_phase / rtl_phase)+ §4.4.1 latch 寄存器 + §4.7 守卫触发标志 — G3 logsout 可观测项候选 |
| [H1 验证场景目录](../H-verification/H1-scenario-catalog.md)(Wave 11)| ⇢ | §4.4.3 不连续转换 5 行 + §4.5 三个 profile 阶段触发 + §4.7.2 G-1..G-5 守卫触发场景 → H1 edge-case 候选场景 |
| [H2 验证指标](../H-verification/H2-metrics.md)(Wave 11)| ⇢ | §4.3 rate / jerk PARAM 锁定的硬上限 → H2 验证 setpoint 不超 PARAM 上限的判据;§4.2 cmd_mask 位组合表 → H2 验证 mode-cmd 状态正确性的判据(per [D6 §4.4.1 truth-table](D6-fms-controller-interface.md))|

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-08 | Wave 8 D4 author | 初稿 — 5 内部块拓扑(§4.1)+ 16 mode 的 cmd_mask + setpoint 映射(§4.2)+ 9 槽 rate/jerk 策略表(§4.3)+ 4 项 bumpless 政策(§4.4)+ 3 profile 算法(§4.5)+ stick conditioning 三件套(§4.6)+ cmd_mask emission 5-step 流水线 + 5 守卫(§4.7)+ A6 reset 协议(§4.8)+ D5/D6/E1 hand-off(§4.9);co-seal batch (D3, D4, D6, E1) 声明 |

## Self-check

- [x] frontmatter 完整,字段值合法
- [x] 上游文档全部存在且 status ≥ reviewed,**或**为本工作项所在 co-seal batch 的同批兄弟(允许同批互相引用,需在 §3 注明 batch 名)— D3 / D6 / E1 标记 `draft, co-seal batch (D3, D4, D6, E1)`,符合 [RULES §6 self-check item 2](../RULES.md);其余上游(架构 v1 / A2..A8 / B1..B3 / D1 / D2)全部 reviewed
- [x] 退出条件逐条复核完成,每条均给出依据(§6 表 2 行覆盖 "mode-cmd_mask + setpoint 映射" + "rate/jerk 限制策略" 两条 D4 退出条件)
- [x] 引用路径全部可点击访问(同 design 目录与架构 v1 全部使用相对路径)
- [x] 不存在 RULES §5 禁则中的内容(无 `.slx` 截图,无可执行 `.m` 代码 — §4.5 / §4.6 仅含 `pseudo` 风格算法骨架伪代码;无 firmware 实现细节复述;无重复 bus / enum / param 字段表)
- [N/A] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记 — **本文件 contract_impact=no(shaper 内部;bus/enum/param schema 在 B 区拥有;cmd_mask 位语义在 D6 拥有;数值常量在 D5 拥有),N/A**
- [N/A] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径 — **本文件不镜像 firmware 契约,仅引用 B 区已镜像的字段名 + D6 位语义,N/A**(§3 依赖中保留 `<pending hash>` 占位以与 batch 兄弟一致)
- [x] 下游影响已沿关系图识别完毕(§7 表覆盖 D3 / D5 / D6 / E1 / F3 / G3 / H1 / H2)
- [x] 文档不超出本工作项范围(无越权设计:不重定义 cmd_mask 位号 / 名 / 数值 — 留 B2;不锁 cmd_mask 位语义 — 留 D6;不设计 Stateflow 状态 / 转换 / 守卫 — 留 D3;不锁 Controller 各环算法 / cascade — 留 E1;不锁数值常量 — 留 D5;只设计 Shaper 内部拓扑 + per-mode mapping + rate/jerk 策略 + bumpless + profile 算法 + cmd_mask emission 流水线)
