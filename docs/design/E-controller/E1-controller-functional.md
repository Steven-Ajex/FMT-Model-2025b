---
work_item: E1
title: Controller 功能设计(启用环路、cmd_mask 裁剪规则、各环职责)
upstream: [架构v1, A3, A4, A6, A7, B1, B2, B3, C1, D1, E2, F1, D3, D4, D6]
contract_impact: no
status: reviewed
authored_at: 2026-05-08
last_reviewed_at: 2026-05-08
reviewer_verdict: pass
---

# E1 Controller 功能设计

## 1. 目的

定义 Controller 模块在 Phase 2(多旋翼闭环切片)**作为整体所做之事**:启用哪些控制环路、给定 `FMS_Out_Bus.cmd_mask` 后如何裁剪环路启用 / 旁路、各环路的功能职责(输入 / 输出 / anti-windup 引擎条件 / validity gating / reset 行为 / 多旋翼 leaf 委托)。本文件与同 batch 兄弟 [D6](../D-fms/D6-fms-controller-interface.md) 共同闭合 FMS↔Controller 接口契约:**D6 锁 `cmd_mask` 位语义,E1 锁基于位语义的环路裁剪规则**。

## 2. 范围

**在范围:**

- Phase 2 多旋翼 Controller 启用的 5 个环路(位置 / 速度 / 姿态 / 角速度 / mixer-leaf)的功能身份与速率
- `cmd_mask` 取值组合 → 环路启用 / 旁路 / 级联 hand-off 的裁剪规则(消费 D6 锁定的位语义)
- 每环功能职责:输入字段(`FMS_Out_Bus` / `INS_Out_Bus` / `CONTROL_PARAM` 引用)/ 输出(下一环 reference 或 actuator)/ anti-windup engage 触发条件(算法 = E3)/ validity gating 意图(细则 = F3)/ reset 行为(细节 = A6)/ 多旋翼 leaf 委托点(细则 = E4)
- 级联 hand-off 语义:外环 bypass 时内环 setpoint 来源、bypass 期间外环内部状态策略、跨级 anti-windup 信号传播功能意图
- cmd_mask 切换时的 bumpless transfer 功能要求(具体公式 = E3 / D4)
- `INS_Out_Bus` 各环消费列(echo A3 §4.8 矩阵的 Controller 列)+ validity-drop 功能意图(F3 owns 完整 fallback 表)
- `Control_Out_Bus` 装配字段集与各字段的功能来源(echo B1 §4.4 schema)
- Controller_init / disarm / failsafe 时的功能行为(echo A6)

**不在范围(由其他工作项处理):**

- 级联拓扑 / shared vs leaf 边界 / 库块清单 — 由 [E2 Controller 结构设计](E2-controller-structural.md) 处理(已 reviewed,Wave 7)
- 控制律方程 / 增益结构 / 前馈公式 / anti-windup 算法 / 滤波器系数 — 由 [E3 各环算法设计](E3-loops-algorithm.md)(Wave 9)处理
- 多旋翼分配矩阵 / 几何 / 电机推力曲线 / leaf 增益数值 — 由 [E4 多旋翼 leaf](E4-multicopter-leaf.md)(Wave 9)处理
- 5 ms 周期内各环执行预算 / 热点禁用项 / LUT / 定点策略 — 由 [E5 性能预算](E5-performance-budget.md)(Wave 10)处理
- `cmd_mask` 每位的位语义(置位含义)— 由 [D6 FMS↔Controller 接口约定](../D-fms/D6-fms-controller-interface.md)(Wave 8 sibling)处理;E1 仅消费 D6 已锁的位语义并给环路裁剪规则
- bus / enum / parameter 字段级 schema — 由 [B1](../B-contracts/B1-bus-inventory.md) / [B2](../B-contracts/B2-enum-inventory.md) / [B3](../B-contracts/B3-parameter-schema.md) 拥有
- INS validity 完整 fallback 表(zero-hold / 持锁 / 切备份 / disable mode) — 由 [F3 INS 消费规则](../F-ins-contract/F3-consumption-rules.md)(Wave 9)处理;E1 仅给"每环 validity-drop 期间的功能意图"
- mode → cmd_mask 映射 / shaper 公式 — 由 [D3](../D-fms/D3-mode-manager.md) / [D4](../D-fms/D4-command-shaper.md)(Wave 8 siblings)处理

## 3. 依赖

| 上游 | 引用位置 | 用途 |
|---|---|---|
| 架构 v1 §7.2 | [架构 v1](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | Controller 顶层职责 — "convert `FMS_Out_Bus` plus `INS_Out_Bus` into actuator commands";"owns cascaded control loops, feed-forward, anti-windup, vehicle-specific mixer/allocation" |
| 架构 v1 §8.4 | [架构 v1](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | Controller boundary — 输入 `FMS_Out_Bus` + `INS_Out_Bus`(契约级);输出 `Control_Out_Bus`;"vehicle-specific mixer/allocation is the deepest leaf" |
| 架构 v1 §12.2 | [架构 v1](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | Controller 内部 7 子模块栈(setpoint decode / position / velocity / attitude / rate / mixer / output assembler) — E1 启用 5 个功能环路对齐其中 §12.2.2 / .3 / .4 / .5 / .6 |
| 架构 v1 §13 | [架构 v1](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | Controller = 5 ms / 200 Hz,internally single-rate;每环 rate = 5 ms |
| A3 §4.5.2 | [A3 模块边界](../A-architecture/A3-module-boundaries.md) | Controller 子模块对 `FMS_Out_Bus` 字段依赖矩阵(setpoint decode / position / velocity / attitude / rate / yaw / mixer 7 子模块的字段消费表)|
| A3 §4.5.3 | [A3 模块边界](../A-architecture/A3-module-boundaries.md) | Controller 子模块对 `INS_Out_Bus` 字段依赖矩阵(per-loop INS 字段 + gate 位)|
| A3 §4.5.5 | [A3 模块边界](../A-architecture/A3-module-boundaries.md) | `Control_Out_Bus` 字段表(motor_cmd / actuator_cmd / passthrough echo / debug)|
| A3 §4.7 | [A3 模块边界](../A-architecture/A3-module-boundaries.md) | cmd_mask 守卫指针(位语义 D6+E1 lock)— A3 已落"位号工作名" + "同帧 Latch RB-05",E1 在此基础上写裁剪规则 |
| A3 §4.8 | [A3 模块边界](../A-architecture/A3-module-boundaries.md) | INS 消费侧依赖矩阵集中表;E1 §4.5 echo Controller 列 |
| A4 §4.4 RB-02 / RB-03 / RB-05 | [A4 跨速率边界](../A-architecture/A4-rate-boundaries.md) | Controller 输入侧 ZOH + Latch(`FMS_Out_Bus` 同帧锁存,`INS_Out_Bus` ZOH);输出 ZOH(`Control_Out_Bus` → Plant 1 ms)|
| A6 §4.2.3 / §4.4.2 / §4.5.2 | [A6 init/reset 契约](../A-architecture/A6-init-reset-contract.md) | Controller_init 必须使其确定的状态(各环 integrator / anti-windup / filter / enable latch / mixer 中间态);`Control_Out_Bus` reset 后稳态(零推力 / trim);reset 协调(集中 FMS 权威)|
| A7 §4.5 | [A7 时间约定](../A-architecture/A7-time-conventions.md) | Controller 内 dt 由固定步长得来,不基于 `INS_Out_Bus.timestamp` 或 step interface timestamp 派生 |
| B1 §4.2 / §4.3.1 / §4.3.4 / §4.4 / §4.7 | [B1 Bus 清单](../B-contracts/B1-bus-inventory.md) | `FMS_Out_Bus` / `INS_Out_Bus` / `Control_Out_Bus` 字段级 schema(E1 引用,不重定义)|
| B2 §4.4.6 / §4.4.14 | [B2 Enum 清单](../B-contracts/B2-enum-inventory.md) | `CtrlMode` 枚举值(E1 用于 ctrl_mode echo);`MASK_BIT_*` 位号工作名清单(POSITION_LOOP=0 / VELOCITY_LOOP=1 / ACCELERATION_LOOP=2 / ATTITUDE_LOOP=3 / RATE_LOOP=4 / YAW_LOOP=5 / YAW_RATE_LOOP=6 / THROTTLE_PASSTHROUGH=7)|
| B3 §4.4 / §4.6 | [B3 Parameter schema](../B-contracts/B3-parameter-schema.md) | `CONTROL_PARAM` 39 字段(E1 引用环路增益 / 限幅 / 滤波 PARAM 字段名;数值由 E4 leaf 锁)|
| C1 §4.x | [C1 Plant 功能设计](../C-plant/C1-plant-functional.md) | Plant fidelity ladder L1..L4(确认 mixer 输出语义与 Plant 输入语义匹配)|
| D1 §4.1 / §4.2 / §4.6 | [D1 FMS 功能设计](../D-fms/D1-fms-functional.md) | 12 主 mode + 4 failsafe sub-mode → ctrl_mode echo;FMS 仲裁后产出的 setpoint 字段集 → E1 消费;§4.8 OQ2 macro 保留约束(FMS 不做内环) |
| E2 §4.1 / §4.2 / §4.5 | [E2 Controller 结构设计](E2-controller-structural.md) | 7 子模块拓扑 + setpoint / measurement / cmd_mask 路由槽位 + 各环内部骨架 — E1 在此骨架上注入裁剪规则 + 功能职责 |
| F1 §4.5 | [F1 ins_stub 功能](../F-ins-contract/F1-ins-stub-functional.md) | INS_Out_Bus 消费侧字段语义(MIL 路径上 stub 行为)|
| **D3**(Wave 8 sibling) | [D3 Mode Manager](../D-fms/D3-mode-manager.md) — *起草中* | mode → ctrl_mode → cmd_mask 映射 — E1 假设 D3 产出的 cmd_mask 与 E1 §4.2 裁剪表覆盖的组合一致;若 D3 引入未在 §4.2 列出的组合,E1 §8 变更日志同步 |
| **D4**(Wave 8 sibling) | [D4 Command Shaper](../D-fms/D4-command-shaper.md) — *起草中* | shaper 产出的 setpoint 在每个 cmd_mask 组合下保证非零字段一致 — E1 §4.2 裁剪表的"setpoint 来源"列假设 D4 已经把对应 setpoint 字段填充正确 |
| **D6**(Wave 8 sibling) | [D6 FMS↔Controller 接口约定](../D-fms/D6-fms-controller-interface.md) — *起草中* | **`cmd_mask` 位语义 + 位间优先级 + 互斥规则 — E1 完全消费**。E1 §4.2 表的"激活环路"列由 D6 §4.x 位语义直接派生;D6 §4.x 互斥规则在 E1 §4.2 引用条目中注明 |
| FMT-Firmware @ `<pending hash>` | `FMT-Firmware/src/model/control/<vehicle>/lib/Controller_types.h` | 契约镜像源(`FMS_Out_Bus.cmd_mask` 字段类型 / `Control_Out_Bus` 字段顺序 — 由 B1 / B5 镜像 ledger 拥有,E1 仅引用)|

**Co-seal batch 注记 — `(D3, D4, D6, E1)` Wave 8:**

本工作项位于 **Wave 8 co-seal batch `(D3, D4, D6, E1)`**(per [00-design-plan §4.E E1 行](../00-design-plan.md):"cmd_mask 裁剪规则部分需与 D6 同 batch 评审(co-seal),整个 E1 推迟到 Wave 8 启动以避免 D6 → E1 强前置违反";per [01-design-relationships §5 entry 2](../01-design-relationships.md):"(D3, D4, D6, E1) — Mode Manager / Command Shaper / cmd_mask 语义 / Controller 功能(含环路裁剪规则)。E1 与 D6 由强前置升级为同 batch")。

D3 / D4 / D6 在本文件起草时**同步处于 draft 状态**,本文件按 [RULES §6 第 2 项](../RULES.md) "或为本工作项所在 co-seal batch 的同批兄弟(允许同批互相引用,需在 §3 注明 batch 名)" 援引。**Wave 8 co-seal 评审时,D3 / D4 / D6 / E1 四份一并核对一致性**:

- D6 提交的 `cmd_mask` 位语义表必须**枚举** E1 §4.2 中每个组合;反之 E1 §4.2 不引入 D6 未枚举的位组合
- D3 输出的 `cmd_mask` 值集合必须**子集** E1 §4.2 表覆盖的组合;反之 E1 §4.2 不为 D3 不会发出的组合写裁剪规则
- D4 的 shaper 在 D3 产出 cmd_mask 的每帧必须**填充** E1 §4.2 该行 "active setpoint" 列声明的字段;反之 E1 不为 D4 未填的字段写消费规则

环境注:FMT-Firmware 在本设计阶段**未挂载**;本文件按 D6 sibling 提供的位语义(co-seal Wave 8 锁定)+ A3 / B1 / B2 引用字段名撰写,不直接读取 firmware 头文件。

## 4. 设计内容

### 4.1 Phase 2 启用的环路

Controller 在 Phase 2(多旋翼)启用 **5 个环路**(对齐 [架构 v1 §12.2](../../architecture/2026-05-05-fmt-model-architecture-v1.md) 子模块 .2 / .3 / .4 / .5 / .6),全部跑在 5 ms / 200 Hz 单速率 internally(架构 v1 §13)。每环的速率 / 输入 / 输出 / 默认 enable 条件如下;具体 `cmd_mask` 组合驱动的 enable / bypass 决策见 §4.2,具体功能职责见 §4.3。

#### 4.1.1 环路总表

| Loop ID | 工作名 | 子模块定位(架构 v1)| 速率 | 主输入 setpoint(`FMS_Out_Bus`)| 主输入 measurement(`INS_Out_Bus`)| 主输出 | 默认 enable 条件(摘要)|
|---|---|---|---|---|---|---|---|
| **L-01** | Position(NED 外环) | §12.2.2 Position/Outer Guidance Loop | 5 ms | `pos_cmd_ned_m[3]` | `position_ned_m[3]` + `flag.position_valid` | 内部 `vel_sp_inner` → 馈入 L-02 | `MASK_BIT_POSITION_LOOP=1` AND not(L-02/L-03/L-04 任一外部 setpoint 已直接给出) |
| **L-02** | Velocity(NED) | §12.2.3 Velocity Loop | 5 ms | `vel_cmd_ned_mps[3]`(use_external)或 L-01 输出(use_inner)| `velocity_ned_mps[3]` + `flag.velocity_valid` | 内部虚拟控制(thrust + tilt acceleration ref)→ 馈入 L-03 / L-05 | `MASK_BIT_VELOCITY_LOOP=1` 或 上游 L-01 active(use_inner)|
| **L-03** | Attitude | §12.2.4 Attitude Loop | 5 ms | `att_cmd_quat[4]` 或 `att_cmd_euler_rad[3]`(use_external)+ `yaw_cmd_rad`(yaw 分量);或 L-02 派生(use_inner)| `quat_ned_to_b[4]` + `flag.attitude_valid` | 内部 `ang_rate_sp_inner[3]` → 馈入 L-04 | `MASK_BIT_ATTITUDE_LOOP=1` 或 上游 L-01/L-02 active(use_inner)|
| **L-04** | Angular rate(body) | §12.2.5 Rate Loop(hot loop) | 5 ms | `ang_rate_cmd_b_radps[3]`(use_external)+ `yaw_rate_cmd_radps`(yaw 分量);或 L-03 派生(use_inner)| `ang_rate_b_radps[3]`(典型 — gate `flag.attitude_valid`)| body-frame moment + thrust virtual control → 馈入 L-05 | `MASK_BIT_RATE_LOOP=1` 或 上游 L-03 active(use_inner)— 典型 always-on 当 cmd_mask ≠ 0 |
| **L-05** | Mixer / Allocator(多旋翼 leaf) | §12.2.6 Mixer/Allocator | 5 ms | L-04 输出(moment + thrust);或 `MASK_BIT_THROTTLE_PASSTHROUGH=1` 时直消费 `FMS_Out_Bus.throttle_cmd` | (无 INS 直消费) | `motor_cmd[]`(发到 §4.6 Output Assembler)| always-on 当 cmd_mask ≠ 0;详细 leaf 内部由 [E4](E4-multicopter-leaf.md) |

**关键不变量:**

1. **5 个环路全部 internally 单速率 = 5 ms**(架构 v1 §13);E1 不引入 sub-rate。
2. **L-05(mixer)是多旋翼 leaf**:E1 仅声明其作为 5 个 active 环路之一存在 + 其输入 / 输出语义边界;**内部分配矩阵 / 几何 / 电机推力曲线**由 [E4](E4-multicopter-leaf.md) 锁定。
3. **`MASK_BIT_ACCELERATION_LOOP=2`(B2 §4.4.14 位号 2)在 Phase 2 默认融合到 L-02 内部**(由 D4 shaper 产出 `acc_cmd_ned_mps2` 作为 L-02 前馈输入,不开 L-02 之外的独立"加速度环");**位号本身保留**为 D6 锁定的桥接信号。详见 §4.2 acc-loop 行。
4. **`MASK_BIT_YAW_LOOP=5` / `MASK_BIT_YAW_RATE_LOOP=6`** 在 Phase 2 不形成"独立环",而是作为 L-03 / L-04 的 **yaw 分量子通道**:`yaw_cmd_rad` 进 L-03 yaw 分量,`yaw_rate_cmd_radps` 进 L-04 yaw 分量;详见 §4.2.6 与 §4.3.3 / §4.3.4。
5. **`MASK_BIT_THROTTLE_PASSTHROUGH=7`** 在 Phase 2 启用时**绕开 L-05 mixer 内部 thrust 计算**,直接以 pilot/FMS 油门驱动 mixer 的 collective 输入;详见 §4.2.7 与 §4.3.5。

#### 4.1.2 环路 enable 与 cmd_mask 的关系总览

`cmd_mask` 是 `FMS_Out_Bus` 的 uint32 bitfield(B2 §4.4.14;B1 §4.7);Controller 在每帧:

1. 同帧 latch `cmd_mask` 与所有 setpoint 字段(per A3 §4.5.2,RB-05 整 bus 同帧锁存);
2. 调用 `Controller_CmdMask_Resolver`(E2 §4.3.2)解析 cmd_mask + ctrl_mode → 每环 `(use_external, use_inner)` 选择信号;
3. 每环按 §4.2 表激活对应行为;非激活环路输出零 / hold-last(per §4.4 hand-off 策略)。

**E1 与 D6 的接口边界**:E1 **消费** D6 §4.x 锁定的:(a) 位语义("第 i 位 = 1 时含义"),(b) 位间优先级("同时置位 X / Y 时谁赢"),(c) 互斥规则("X 与 Y 不可同时置位")。E1 在此之上**产出** "cmd_mask 取值组合 → 5 环 active / bypass 状态"的查表(§4.2)。

### 4.2 cmd_mask 裁剪规则(co-seal closure with D6)

#### 4.2.1 总表(cmd_mask 组合 → 环路裁剪)

下表枚举 Phase 2 多旋翼下 D1 / D3 在 §4.1.1 12 主 mode + 4 failsafe sub-mode 中可能产出的 `cmd_mask` 组合(对应 [D1 §4.6.2 + §4.1.1 mode 表](../D-fms/D1-fms-functional.md))。**位语义引自 D6 §4.x**(co-seal sibling);**位间优先级 / 互斥引自 D6 §4.x**;E1 在此基础上加 "outer-most active loop / 激活环路 / 旁路环路 / setpoint 来源" 四列。

| # | cmd_mask 组合(B2 §4.4.14 位号工作名)| Outer-most active loop | 激活环路(参与计算)| 旁路环路 | setpoint 来源(per 激活环)| 典型 mode(D1 §4.1.1) | D6 引用 |
|---|---|---|---|---|---|---|---|
| C-01 | `POSITION_LOOP` only(bit 0=1)| L-01 | L-01, L-02, L-03, L-04, L-05 | none | L-01 ← `pos_cmd_ned_m`;L-02..L-04 ← inner cascade;L-05 ← L-04 输出 | M-05 POSHOLD / M-06 TAKEOFF / M-09 LOITER / M-10 MISSION / M-11 OFFBOARD-position / M-08 RTL | D6 §4.3 priority Table 行 1(POS dominates) |
| C-02 | `POSITION_LOOP` + `VELOCITY_LOOP`(bits 0+1=1)| L-01 | L-01, L-02, L-03, L-04, L-05 | none | 同 C-01;`vel_cmd_ned_mps` 作为 L-02 **前馈**(非主 setpoint)| M-05/M-06/M-08 含 D4 vel 前馈;M-11 OFFBOARD pos+vel | D6 §4.3(POS+VEL 共置位 = POS 主 + VEL ff) |
| C-03 | `VELOCITY_LOOP` only(bit 1=1)| L-02 | L-02, L-03, L-04, L-05 | L-01 | L-02 ← `vel_cmd_ned_mps`(direct);L-03..L-04 ← inner;L-05 ← L-04 | M-04 ALTHOLD(vz)/ M-07 LAND(vz)/ M-11 OFFBOARD-velocity | D6 §4.3 行 2(VEL without POS) |
| C-04 | `VELOCITY_LOOP` + `YAW_LOOP`(bits 1+5=1)| L-02 | L-02, L-03(仅 yaw)+ L-03 (roll/pitch from L-02 派生),L-04, L-05 | L-01 | L-02 ← `vel_cmd_ned_mps`;L-03 yaw 分量 ← `yaw_cmd_rad`(direct);L-03 roll/pitch 分量 ← L-02 派生;L-04 ← L-03 inner;L-05 ← L-04 | M-05 POSHOLD with yaw hold(velocity-mode 派生)/ M-08 RTL face-home | D6 §4.3(yaw absolute compatible with velocity) |
| C-05 | `ATTITUDE_LOOP` + `THROTTLE_PASSTHROUGH`(bits 3+7=1)| L-03 | L-03, L-04, L-05(throttle 直驱 collective)| L-01, L-02 | L-03 ← `att_cmd_quat` 或 `att_cmd_euler_rad`(direct);L-04 ← L-03 inner;L-05 collective ← `throttle_cmd`(direct, bypass 内部 thrust 计算)| M-03 STABILIZE | D6 §4.3(attitude + manual throttle 互不互斥) |
| C-06 | `ATTITUDE_LOOP` only(bit 3=1)| L-03 | L-03, L-04, L-05 | L-01, L-02 | L-03 ← `att_cmd_quat`;L-04 ← L-03 inner;L-05 ← L-04 输出(含 mixer-内 thrust)| (rare;若 D4 不发 throttle_passthrough)| D6 §4.3 行 3 |
| C-07 | `ATTITUDE_LOOP` + `YAW_RATE_LOOP` + `THROTTLE_PASSTHROUGH`(bits 3+6+7=1)| L-03 | L-03(roll/pitch)+ L-04(yaw rate 直驱), L-05 | L-01, L-02, L-04 roll/pitch | L-03 roll/pitch ← `att_cmd_quat`(roll/pitch 分量);L-04 yaw 分量 ← `yaw_rate_cmd_radps`(direct);L-05 collective ← `throttle_cmd` | M-03 STABILIZE 增强(yaw stick rate)| D6 §4.3(L-04 partial direct)|
| C-08 | `RATE_LOOP` + `THROTTLE_PASSTHROUGH`(bits 4+7=1)| L-04 | L-04, L-05 | L-01, L-02, L-03 | L-04 ← `ang_rate_cmd_b_radps`(direct);L-05 collective ← `throttle_cmd` | M-02 MANUAL(rate)/ M-12 ACRO | D6 §4.3 行 4(acro-style)|
| C-09 | `RATE_LOOP` only(bit 4=1)| L-04 | L-04, L-05 | L-01, L-02, L-03 | L-04 ← `ang_rate_cmd_b_radps`;L-05 ← L-04 输出(含 mixer-内 thrust)| (rare diagnostic)| D6 §4.3 |
| C-10 | All-zero(`cmd_mask == 0`) | none | none(全部环路 bypass) | L-01..L-05 | (无 setpoint 消费)| M-01 DISARMED / FS-04 DISARM / Controller_init 后 / `Plant_States_Bus` invalid | D6 §4.7(safe state)|
| C-11 | `ACCELERATION_LOOP`(bit 2=1)+ 任一上层组合 | depends on co-set bits | acc_cmd_ned_mps2 进 L-02 前馈 | depends | acc_cmd_ned_mps2 进 L-02 `FeedForward_Composer` 输入(不形成独立 acc 环)| M-05/M-06/M-08/M-11 D4 jerk-limited 前馈 | D6 §4.4(acc 必须 co-set with POS 或 VEL)|

**§4.2.1 解读规则(应对 D6 互斥决议):**

- **D6 锁定的位优先级**(per D6 §4.3 priority table)决定多位同时置位时的"outer-most active loop":若同时置位 POS+VEL → POS 赢(C-02);POS+ATT → POS 赢(C-01 等价,VEL/ATT 走 inner);ATT+RATE → ATT 赢(C-06 等价,RATE 走 inner)。
- **D6 锁定的互斥规则**(per D6 §4.4 mutex rules)定义非法组合;Controller 收到非法 cmd_mask 时按 D6 §4.4 决议(典型:走 §4.2.2 fallback to all-zero)。
- **D6 §4.x 余下未在上表覆盖的组合**:E1 §4.2.2 给"未覆盖组合"通用规则。

#### 4.2.2 未覆盖组合的通用裁剪规则

若 D6 锁定的位语义产生 §4.2.1 表外的 cmd_mask 组合(D3 后续可能引入 / 或外部 GCS / Auto 流注入),Controller 按以下规则:

1. **优先级解释**:按 D6 §4.3 优先级,从高到低识别 outer-most active bit;outer-most active = 该 bit 对应的环路。
2. **激活环路 = outer-most active loop + 其下所有内环 + L-05 mixer**:cascade 默认 "上游 active → 下游也 active(use_inner)"。
3. **旁路环路 = outer-most active 之外的所有外环**:bypass 状态见 §4.4.1。
4. **setpoint 来源**:outer-most active 取 `use_external`(直接消费 `FMS_Out_Bus` 对应字段);其下内环取 `use_inner`(消费上游环输出)。
5. **若所有 active bit 全部为 throttle_passthrough(bit 7) + 其他 inner(bit 4)**:走 C-08 acro-style。
6. **若仅 throttle_passthrough(bit 7) 单置位,无任何环路 bit**:**E1 视作非法**(物理上 mixer 需要 attitude 至少稳定);走 §4.2.3 illegal-combo fallback;由 D6 §4.4 互斥规则锁定该组合是否合法。
7. **若 cmd_mask == 0**:走 C-10(disarm-equivalent;Controller 输出 A6 §4.4.2 安全态)。

#### 4.2.3 非法 cmd_mask 组合的 fallback(per D6 §4.4 引用)

D6 §4.4 给出"互斥规则"和"非法组合检测",当 Controller 收到 D6 标为非法的 cmd_mask:

- **Phase 2 默认行为**:Controller **不**自行修改 cmd_mask;按 §4.2.1 C-10 行为(全部环路 bypass + Controller 输出 A6 §4.4.2 安全态:零推力 / trim);并 echo `cmd_mask` 到 `Control_Out_Bus.cmd_mask`(per A3 §4.5.5),让 Plant trace / log 可追溯非法。
- **错误 telemetry**:`Control_Out_Bus.debug_*` 字段(若 B1 锁定存在)标记 "illegal_cmd_mask=true";具体字段名由 B1 / E5 锁。
- **不向 FMS 反馈**:Controller 不修改 `FMS_Out_Bus`(per A3 §4.5 单向);非法组合的最终修正责任在 FMS Safety Monitor(D2)或 D3 Mode Manager(下一帧产出合法 cmd_mask)。

#### 4.2.4 cmd_mask 切换时的 bumpless transfer(功能要求)

cmd_mask 在帧 t-1 → t 切换时(例:POSHOLD → ALTHOLD 由 D3 产出 mask 改变,POSITION_LOOP 位 1→0、VELOCITY_LOOP 位 0→1),为避免 setpoint 跳变导致的输出阶跃,E1 要求:

1. **outer-most active 改变时**(§4.2.1 行切换),新 outer-most active 环路的 setpoint 在第 t 帧由其 **`use_external` 路径**驱动(直接消费 FMS 对应字段)— D4 shaper 必须保证该字段在切换帧已被填充并连续(per D4 sibling)。
2. **被新 bypass 的旧外环**:其内部状态按 §4.4.2 处理(zero-frozen 或 hold-last,功能策略见 §4.4.2)。
3. **新激活的"中间环"使用内 cascade**:其 setpoint 在第 t 帧由新 outer-most 环输出驱动(use_inner);**功能要求是该 setpoint 不出现可观测阶跃** — 实现层(E3)通过(a) 切换帧重新初始化新外环 integrator 到使其 inner setpoint 等于切换前内环测量值,或(b) D4 sibling 在 setpoint 上直接做 rate-limited 渐变。E1 仅声明 "bumpless transfer 是功能要求";具体公式 / 实现选 (a) vs (b) 由 E3 / D4 锁定。
4. **anti-windup 状态**:cmd_mask 切换时,新 bypass 环路的 integrator 立即 freeze(stop accumulating);新激活环路的 integrator 按 §4.4.4 reset 策略处理。
5. **disarm → arm 切换**(C-10 → C-01..C-09):走 §4.7 reset 路径(全 integrator 清零,§4.7 等价 init 后稳态)。

### 4.3 各环功能职责

每环以下列出:**(a) 输入字段(`FMS_Out_Bus` / `INS_Out_Bus` / `CONTROL_PARAM` 引用)**;**(b) 输出**;**(c) anti-windup engage 触发条件(算法 = E3)**;**(d) validity gating 功能意图(F3 锁完整 fallback 表)**;**(e) reset 行为(细节 = A6)**;**(f) 多旋翼 leaf 委托点(细节 = E4)**。

#### 4.3.1 L-01 Position loop(NED 外环)

**(a) 输入:**

- setpoint:`FMS_Out_Bus.pos_cmd_ned_m[3]`(per A3 §4.5.2 + B1 §4.7;FMS gated by `MASK_BIT_POSITION_LOOP`)
- measurement:`INS_Out_Bus.position_ned_m[3]`(per A3 §4.8 矩阵 Ctrl Pos 列 = ●);`flag.position_valid` 作为 gate(per A3 §4.8 = gate)
- params:`CONTROL_PARAM` 中位置环增益子集(per [B3 §4.4.6](../B-contracts/B3-parameter-schema.md);具体字段名由 B3 ledger 锁,典型类别:`pos_p_gain_xy` / `pos_p_gain_z` / `pos_xy_lim_mps` / `pos_z_up_lim_mps` / `pos_z_down_lim_mps`;数值由 [E4](E4-multicopter-leaf.md) leaf 注入)

**(b) 输出:**

- `vel_sp_inner[3]`(NED frame):内部 wire,馈入 L-02 `setpoint_inner` port(per E2 §4.5.1 输出端口)
- 不直接产出 `Control_Out_Bus` 任何字段
- saturation flag:输出限幅命中标志,反馈给 anti-windup(per (c))

**(c) Anti-windup 行为(功能层;算法 = E3):**

- engage 条件:L-01 输出 `vel_sp_inner` 达到 limit(`pos_xy_lim_mps` / `pos_z_*_lim_mps`)且 integrator 仍在累加 setpoint-measurement 误差与 limit 同号
- 保存状态:integrator 状态被 freeze(不累加);具体 freeze 策略(back-calc vs conditional integration)= E3 锁
- 不持有 `cmd_mask` 切换时的 anti-windup 状态(由 §4.4.4 reset 策略覆盖)

**(d) Validity gating(功能意图;F3 锁完整 fallback):**

- `flag.position_valid == 0` 时 L-01 **不产生**有效 `vel_sp_inner`;**功能意图** = "把 L-01 整体视作 bypass 状态(等价 §4.4.1 旁路语义),内环 L-02 改用 `use_external` 路径(若 cmd_mask `VELOCITY_LOOP` 也置位)或继续 `use_inner` 但消费 zero / hold-last(由 §4.4.2)"。
- gate 信号 `flag.position_valid` 在 §4.5 路由表中接 §4.1.1 解码侧 `valid_gate` 端口(E2 §4.7)。
- **完整 fallback 表(zero-out / hold-last / 切备份估计 / 触发 FMS-side TR-04 GPS 失效)由 [F3](../F-ins-contract/F3-consumption-rules.md) 锁定**;E1 仅声明上述意图。

**(e) Reset 行为(per A6):**

- `Controller_init` 完成后:integrator state = 0(per A6 §4.2.3 Controller 项 1);saturation latch = false;`vel_sp_inner` 输出 = 0 向量
- `FMS_Out_Bus.reset` 上拉(per A6 §4.5.2 集中 FMS 权威):立即清零 integrator + saturation latch + 输出 = 0 向量(等价 init 后稳态;A6 §4.4 默认 "reset 后稳态 = init 后稳态")
- **跨 cmd_mask 切换的状态保持策略**:见 §4.4.2(L-01 bypass 时 zero-frozen vs hold-last);默认 zero-frozen
- mode 切换(per A6 §4.3.1 TS-MODE-XCHG):由 D3 触发 `FMS_Out_Bus.reset` 实现 global reset;Controller 不自行解释 ctrl_mode 跳变(per A6 §4.5.2 协议)

**(f) 多旋翼 leaf 委托点(per E4):**

- 增益数值集 → [E4](E4-multicopter-leaf.md);E1 仅引用 B3 字段名
- 输出限幅常数 → E4
- 单位约定 NED-m / NED-mps:遵 [A2 §4.10 单位 / 坐标系](../A-architecture/A2-naming-conventions.md);E1 不重述

#### 4.3.2 L-02 Velocity loop(NED)

**(a) 输入:**

- setpoint:**`use_external` 路径**消费 `FMS_Out_Bus.vel_cmd_ned_mps[3]`(per A3 §4.5.2,gated by `MASK_BIT_VELOCITY_LOOP`);**`use_inner` 路径**消费 L-01 输出 `vel_sp_inner`
- 加速度前馈(可选):`FMS_Out_Bus.acc_cmd_ned_mps2[3]`(gated by `MASK_BIT_ACCELERATION_LOOP`,per §4.1.1 不变量 3:Phase 2 acc 走 L-02 前馈不开独立环)
- measurement:`INS_Out_Bus.velocity_ned_mps[3]`(A3 §4.8 矩阵 ●);`flag.velocity_valid` gate
- params:`CONTROL_PARAM` 速度环增益子集(B3;典型 `vel_pi_gain_xy` / `vel_pi_gain_z` / `acc_xy_lim_mps2` / `tilt_lim_rad`;数值 = E4)

**(b) 输出:**

- 内部虚拟控制 = thrust 命令(scalar)+ tilt acceleration reference(NED-acc 派生 → 转 body-frame attitude reference)
- 馈入 L-03 `setpoint_inner` port(attitude 路径;具体 NED-acc → quat 转换 = E3)
- 馈入 L-05 collective(thrust 路径;若 `MASK_BIT_THROTTLE_PASSTHROUGH` 不置位)
- saturation flags 反馈给 anti-windup

**(c) Anti-windup:**

- engage:tilt limit(`tilt_lim_rad`)或 thrust limit(`thrust_min` / `thrust_max`,由 E4 leaf 注入)饱和
- E3 锁算法(典型 back-calc 协调 tilt + thrust 同时饱和)

**(d) Validity gating:**

- `flag.velocity_valid == 0` 时 L-02 输出由 §4.4.2 策略决定;典型 zero-frozen → L-03 / L-05 改用 hold-last 或上游 mode 退化(D1 TR-04 / TR-08)
- F3 锁完整 fallback

**(e) Reset:**

- `Controller_init`:integrator = 0;saturation latch = false;输出 = 0
- `FMS_Out_Bus.reset` 拉:同上
- cmd_mask 切换 bumpless:per §4.2.4(若新 outer-most 是 L-02,本环 integrator 按 §4.4.4 (b) 初始化)

**(f) Leaf 委托:**

- 增益 / 限幅数值 → E4
- NED-acc → body-frame tilt 解算的"基础参考方向(g)"参数 → E4(若引入 mass / hover thrust 校正,字段名 B3 锁)

#### 4.3.3 L-03 Attitude loop

**(a) 输入:**

- setpoint:**`use_external`** 消费 `FMS_Out_Bus.att_cmd_quat[4]` 或 `att_cmd_euler_rad[3]`(B1 锁二选一 — per [B1 §4.7](../B-contracts/B1-bus-inventory.md);gated by `MASK_BIT_ATTITUDE_LOOP`);**`use_inner`** 消费 L-02 派生 attitude reference
- yaw 子通道:`FMS_Out_Bus.yaw_cmd_rad`(gated by `MASK_BIT_YAW_LOOP`)— 与 quat / euler 主通道协同(典型:roll/pitch 来自 quat,yaw absolute 来自 yaw_cmd 单字段)
- measurement:`INS_Out_Bus.quat_ned_to_b[4]`(A3 §4.8 ●);`flag.attitude_valid` gate
- params:`CONTROL_PARAM` 姿态环增益(典型 `att_p_gain_xy` / `att_p_gain_z` / `tilt_lim_rad` / `att_yaw_p_gain`;E4 数值)

**(b) 输出:**

- `ang_rate_sp_inner[3]`(body-frame):馈入 L-04 `setpoint_inner` port

**(c) Anti-windup:**

- L-03 典型为 P-only 或 PD(无 integrator 累加)— anti-windup 需求弱;若 E3 引入 P+I,engage 条件 = `ang_rate_sp_inner` 达到 L-04 导出的 rate limit 时 freeze

**(d) Validity gating:**

- `flag.attitude_valid == 0` 时 L-03 不产生有效 `ang_rate_sp_inner`;**功能意图** = critical 失效(姿态环是稳定性根基) — 触发 D1 TR-03 INS 失效(F3 锁联动)
- F3 完整 fallback

**(e) Reset:**

- `Controller_init`:integrator(若有)= 0;`ang_rate_sp_inner` 输出 = 0;quat error 初始化为 identity(per E2 §4.1 表行 §4.1.4)
- `FMS_Out_Bus.reset` 拉:同上

**(f) Leaf 委托:**

- 增益 → E4
- yaw absolute vs yaw heading 的"yaw shortest-path"语义 → E4(若有变体)

#### 4.3.4 L-04 Angular rate loop(body-frame;hot loop)

**(a) 输入:**

- setpoint:**`use_external`** 消费 `FMS_Out_Bus.ang_rate_cmd_b_radps[3]`(gated by `MASK_BIT_RATE_LOOP`);**`use_inner`** 消费 L-03 输出 `ang_rate_sp_inner`
- yaw rate 子通道:`FMS_Out_Bus.yaw_rate_cmd_radps`(gated by `MASK_BIT_YAW_RATE_LOOP`)— 典型与主通道 z 分量替换(per D6 §4.x co-seal 锁定)
- measurement:`INS_Out_Bus.ang_rate_b_radps[3]`(A3 §4.8 ●);gate `flag.attitude_valid`(per A3 §4.5.3 注:角速率环依赖 INS filtered ω,典型用 attitude_valid 同 gate)
- params:`CONTROL_PARAM` 角速率环增益(典型 `rate_pid_gain_xy` / `rate_pid_gain_z` / `rate_d_lpf_cutoff_hz` / `motor_cmd_lim_*`;E4 数值)

**(b) 输出:**

- body-frame moment(3-vec)+ thrust virtual control(scalar,若 L-02 不直接产出 thrust)
- 馈入 L-05 mixer

**(c) Anti-windup(hot loop,关键):**

- engage:motor cmd 在 L-05 mixer 输出端饱和(`motor_cmd[]` 触限);saturation 信号经 `Controller_AntiWindup_Coordinator`(E2 §4.3.2)从 L-05 → L-04 反馈
- L-04 integrator freeze + I 项 back-calc(算法 = E3)
- D-term 在 measurement 端用 `LowPass_Filter_1st`(A8 §4.2.2;cutoff PARAM = `rate_d_lpf_cutoff_hz`)滤波,避免 measurement 噪声放大

**(d) Validity gating:**

- `flag.attitude_valid == 0` → L-04 measurement 不可信;**功能意图** = critical 失效(rate loop 是 mixer 直接前置) — 触发 D1 TR-03 + Controller 输出零推力(等价 §4.2.3 illegal fallback)
- F3 完整 fallback(rate 环可能允许 sensor-direct fallback,若 INS 提供 raw IMU 通道 — 但 Phase 2 默认不开,per A3 §4.5.3 注)

**(e) Reset:**

- `Controller_init`:integrator = 0;D-term LPF state = 0(per A6 §4.2.3 Controller 项 2);输出 = 0
- `FMS_Out_Bus.reset`:同上

**(f) Leaf 委托:**

- 增益 / D-term cutoff / output 限幅 → E4
- `motor_cmd[]` 编码语义(PWM_us vs 0..1 normalized)→ E4 + B4 contract diff 联动锁(A3 §4.5.5 已标 deferred)

#### 4.3.5 L-05 Mixer / Allocator(多旋翼 leaf)

**(a) 输入:**

- L-04 输出:body-frame moment + thrust virtual control
- **替代 thrust 路径**:若 `MASK_BIT_THROTTLE_PASSTHROUGH` 置位,`FMS_Out_Bus.throttle_cmd` 直接驱动 collective(bypass L-04 thrust 计算)
- 分配矩阵 / 几何 / 电机推力曲线参数 → **全部 E4 leaf 注入**(per E2 §4.3.3)
- params:`CONTROL_PARAM` 多旋翼几何子集(典型 `motor_count` / `arm_length_m` / `motor_kf` / `motor_km` / `mixer_matrix[]`;数值 = E4)

**(b) 输出:**

- `motor_cmd[]`(N-vec,N = motor_count;典型 4 / 6 / 8;Phase 2 多旋翼默认 N=4 quad-X)
- 馈入 §4.6 Output Assembler

**(c) Anti-windup(L-05 端的 saturation 上传):**

- L-05 `motor_cmd[]` 触限 → saturation flag 上传 `Controller_AntiWindup_Coordinator` → L-04 / L-02 / L-01 anti-windup engage(E3 锁跨级算法)

**(d) Validity gating:**

- L-05 不直接消费 INS 字段;但若上游 L-02 / L-04 因 INS 失效而输出 zero / hold-last,L-05 接收到的 input 已经反映 fallback

**(e) Reset:**

- `Controller_init`:`motor_cmd[]` = 零推力(per A6 §4.4.2 `Control_Out_Bus` 行 "电机推力 / PWM 命令 = HARDCODED 零推力 / disarmed PWM";具体数值 E4 锁,典型 normalized = 0 或 PWM = `motor_disarm_pwm_us` PARAM)
- `FMS_Out_Bus.reset`:同上

**(f) Leaf 委托:**

- 全部内部细节 → [E4](E4-multicopter-leaf.md):分配矩阵 / 几何 / 电机推力曲线 / 输出归一化 / 限幅常数 / disarm PWM 数值

### 4.4 级联 hand-off 语义

#### 4.4.1 外环 bypass 时内环的 setpoint 来源(per §4.2.1 表)

| Cmd_mask 组合(§4.2.1)| outer-most active | bypass 外环 | 内环 setpoint 路径 |
|---|---|---|---|
| C-01 / C-02(POS active)| L-01 | none | L-02 ← L-01 输出(use_inner);L-03 ← L-02 派生 attitude;L-04 ← L-03 输出 |
| C-03 / C-04(VEL active 无 POS)| L-02 | L-01 | L-02 ← `vel_cmd_ned_mps`(use_external);L-03 ← L-02 派生;L-04 ← L-03 输出 |
| C-05 / C-06 / C-07(ATT active 无 POS / VEL)| L-03 | L-01, L-02 | L-03 ← `att_cmd_*`(use_external);L-04 ← L-03 输出(use_inner)|
| C-08 / C-09(RATE active 无 ATT)| L-04 | L-01, L-02, L-03 | L-04 ← `ang_rate_cmd_b_radps`(use_external);L-05 ← L-04 输出 |
| C-10(all-zero)| none | all | (无 setpoint;Controller 输出 A6 §4.4.2 安全态)|

**关键不变量:**

- Cascade hand-off 的 single-source 原则:每个 active 环路恰有一个 setpoint 源(`use_external` XOR `use_inner`),由 `Controller_CmdMask_Resolver`(E2 §4.3.2)单点解析,environment 内不重复。
- L-05 mixer 始终在 active 环路链末端;若 cmd_mask = 0(C-10),mixer 输出零推力但仍被 Output Assembler 消费(避免下游字段 invalid)。

#### 4.4.2 Bypass 期间外环内部状态策略

**E1 默认策略 = "zero-frozen + integrator clear"**:

- bypass 环路的 integrator state = 0(立即清零,**不**累加);
- bypass 环路的 saturation latch = false;
- bypass 环路的 输出 = 0 向量(便于下游 use_inner 在切换瞬间得到确定的 zero 而非脏值);
- bypass 环路的 LPF / D-term filter state = freeze(不更新,但保留上次值;避免下次 active 时从 0 启动产生瞬态)。

**为什么不选 hold-last**:

- hold-last 在 cmd_mask 切回 active 时会让 integrator 携带过期累积量,违反 bumpless transfer(§4.2.4);
- zero-frozen + 切回 active 时按 §4.4.4 reset 策略重新初始化,语义清晰;
- 与 A6 §4.4.2 "Controller 内部 integrator default ZERO"一致(reset 后稳态默认全 0)。

**例外**:LPF / D-term filter state 默认 freeze 而非 zero — 因为 filter 是无累积量的"惯性环节",freeze + 下次直接消费 measurement 不会产生 windup;且避免每次 cmd_mask 切换都引入 filter transient。

#### 4.4.3 跨级 anti-windup 信号传播(功能意图)

cmd_mask 任何组合下,下游环路 / mixer 饱和都需要让上游 active 环路的 integrator engage anti-windup。E1 的功能意图:

- **Saturation 信号上传**:L-05 `motor_cmd` 触限 → `Controller_AntiWindup_Coordinator`(E2 §4.3.2)广播 saturation flag → 所有当前 active 环路的 anti-windup 单元 engage;
- **bypass 环路不参与**:bypass 环路的 anti-windup 单元忽略 saturation 信号(因为 integrator 已 zero-frozen,无 windup 风险);
- **跨环 freezing 优先级**:同时多环 active 时,inner-most 环 anti-windup 先 engage(直接饱和源),outer-most 环按 cascade 关系延迟 engage(通过 inner setpoint 限幅传播);具体算法(集中协调 vs 分布式)= [E3](E3-loops-algorithm.md);
- **L-04 D-term 在 anti-windup engage 时的行为**:E3 锁(典型 freeze D-term 不再增加,避免饱和期间 D 项放大);E1 仅声明意图。

#### 4.4.4 cmd_mask 切换时 integrator 的 reset 策略(per §4.2.4 bumpless 要求)

- **新激活环路(从 bypass → active)**:
  - 若 cmd_mask 切换源于 D3 mode 切换(per A6 §4.3.1 TS-MODE-XCHG),典型 D3 同时拉 `FMS_Out_Bus.reset` → 全 Controller integrator 清零(等价 init 后稳态)→ **简单 + 安全**,默认走此路径;
  - 若 cmd_mask 切换无伴随 reset(细粒度调整,例:D3 不切 mode 但 D4 改 cmd_mask 启用 acc 前馈),新 active 环路 integrator 由 §4.2.4 (a) 路径在切换帧重新初始化,使其 inner setpoint = 切换前内环 measurement(无阶跃)。
- **新 bypass 环路(从 active → bypass)**:integrator 立即清零(per §4.4.2);saturation latch reset。
- **持续 active 环路(切换前后均 active)**:integrator 不动;只是 setpoint 来源(use_external / use_inner)按 §4.4.1 重新解析。

### 4.5 INS_Out_Bus 消费(echo A3 §4.8 Controller 列)

E1 echo [A3 §4.8 INS 消费侧依赖矩阵](../A-architecture/A3-module-boundaries.md) 的 Controller 列(完整集中表在 A3,本节给 Controller 视角):

| `INS_Out_Bus` 字段(B1 §4.3.1)| L-01 Pos | L-02 Vel | L-03 Att | L-04 Rate | L-05 Mixer | gate 位 |
|---|---|---|---|---|---|---|
| `position_lla.lat_deg / lon_deg / alt_m` | — | — | — | — | — | (FMS 用 — A3 §4.8) |
| `position_ned_m[3]` | ● | — | — | — | — | `flag.position_valid` |
| `velocity_ned_mps[3]` | — | ● | — | — | — | `flag.velocity_valid` |
| `quat_ned_to_b[4]` | — | — | ● | — | — | `flag.attitude_valid` |
| `euler_ned_to_b_rad[3]` | — | — | (alt) | — | — | `flag.attitude_valid`(若 B1 锁 quat 为主则 euler 仅 log)|
| `ang_rate_b_radps[3]` | — | — | — | ● | — | `flag.attitude_valid`(典型;A3 §4.5.3 注)|
| `acc_b_mps2[3]` | — | (opt) | — | — | — | `flag.attitude_valid`(若 E3 引入 mass-feedforward;Phase 2 默认不引入)|
| `INS_Status.ready` | (gate via FMS echo per A3 §4.4.6.5) | (同) | (同) | (同) | — | gate(via FMS) |
| `flag.position_valid` | gate | — | — | — | — | (本身)|
| `flag.velocity_valid` | — | gate | — | — | — | (本身)|
| `flag.attitude_valid` | — | — | gate | gate | — | (本身)|
| `flag.gps_valid` | — | — | — | — | — | (FMS 用 TR-04;Controller 不直接消费)|
| `flag.mag_valid` | — | — | — | — | — | (FMS 用)|
| `timestamp` | — | — | — | — | — | (Controller 不消费 timestamp,per A7 §4.5)|

记号:`●` 直接消费;`gate` 守卫;`(opt)` 可选(Phase 2 默认关);`(alt)` 与 quat 二选一;`—` 不消费。

**Validity-drop 功能意图(per F3 forward-cite):**

| 字段 gate | drop 时(gate==0)的 Controller 功能意图 | 完整 fallback 表 |
|---|---|---|
| `flag.position_valid` | L-01 视作 bypass(等价 §4.4.1 旁路);若 cmd_mask 仍要求 POS active → L-02 退化到 hold-last 或触发 FMS-side TR-04 | F3 |
| `flag.velocity_valid` | L-02 输出按 §4.4.2 zero-frozen;cascade 上游若是 L-01 则 L-01 输出无效;严重情况触发 FMS TR-03 | F3 |
| `flag.attitude_valid` | **critical**(姿态 = 稳定性根基) — L-03 / L-04 双失;Controller 输出零推力(等价 §4.2.3 illegal fallback);触发 FMS TR-03 INS validity loss | F3(严重级)|

**E1 与 F3 的契约边界:** E1 在每环入口 `valid_gate` 端口暴露 gate 信号(per E2 §4.7);**功能意图 = "drop 时停止有效输出"**;**具体 fallback 数值策略**(zero / hold-last / 切备份估计 / disable mode / 触发 FMS event)由 F3 在 §4.x 锁定单一权威表。E1 不重述 F3 表。

### 4.6 输出契约(`Control_Out_Bus`,echo B1 §4.4 schema)

E1 echo [B1 §4.4 `Control_Out_Bus` schema](../B-contracts/B1-bus-inventory.md) + [A3 §4.5.5](../A-architecture/A3-module-boundaries.md);本节给"Controller 视角的功能产出 inventory",**不重定义字段**。

| `Control_Out_Bus` 字段 | 功能来源 | E1 视角的产出方 | 备注 |
|---|---|---|---|
| `motor_cmd[]` | L-05 mixer 输出(经 `Controller_Output_Saturator_Shell` E2 §4.3.2)| L-05 | 多旋翼 N=4 默认;编码 PWM_us vs 0..1 由 E4 + B4 contract diff 联动锁(A3 §4.5.5) |
| `thrust_cmd`(scalar)*(若 B1 锁存在)* | L-04 thrust virtual control 或 §4.6.4 替代路径 | L-04 / L-02 | 若 B1 锁定 `Control_Out_Bus` 含 scalar `thrust_cmd`,E1 在 L-04 输出端装配;若不含,跳过 |
| `actuator_cmd[]` *(passthrough)* | `FMS_Out_Bus.actuator_cmd[]` 直 passthrough(per A3 §4.6 passthrough-with-shape;FMS 侧已整形)| Output Assembler | Controller **不**对 actuator_cmd 做任何修改 / 限幅 / 整形(per A3 §4.5;E2 §4.6) |
| saturation flags(`motor_sat` / `rate_sat` / 各环 sat,B1 锁字段名)| 各环路 saturation latch(E2 §4.3.2 各环 `Saturation_*` 块)| L-01..L-05 + Coordinator | E5 / E1 联动决定字段保留与否(Phase 2 默认保留作 debug 通道) |
| debug rate / debug attitude(若 B1 锁存在)| 各环 setpoint / measurement / error tap | 各环 | E5(Wave 10)决定保留 / 裁剪 |
| `cmd_mask` *(echo)* | `FMS_Out_Bus.cmd_mask` passthrough(A3 §4.5.5)| Output Assembler | Controller 不修改;Plant trace 用 |
| `ctrl_mode` *(echo)* | `FMS_Out_Bus.ctrl_mode` passthrough | Output Assembler | 同上 |
| `throttle_cmd` *(echo)* | `FMS_Out_Bus.throttle_cmd` passthrough(throttle_passthrough mode 时)| Output Assembler | A3 §4.5.5 |
| `reset` *(echo, optional)* | `FMS_Out_Bus.reset` passthrough(若 B1 锁定该字段存在)| Output Assembler | A3 §4.5.5 |
| `timestamp` | step interface 入参(per A7 §4.5)| Output Assembler | Controller 不基于 timestamp 派生 dt |

**装配集中点原则**:per E2 §4.9 + 架构 v1 §12.2.7,所有 `Control_Out_Bus` 字段在**单个** Output Assembler 块内汇集(E2 §4.1.7);Controller 内部不存在多处直写。

### 4.7 Reset / disarm / failsafe 行为

#### 4.7.1 `Controller_init` 完成后的状态(echo A6 §4.2.3)

- 全部 5 个环路(L-01..L-05)integrator state = 0(per A6 §4.2.3 Controller 项 1);
- 全部 LPF / D-term filter state = 0(per A6 §4.2.3 Controller 项 2);
- 全部环路 enable / disable latch = false(per A6 §4.2.3 Controller 项 3 — "默认 disable,直到 FMS 通过 cmd_mask 启用");
- L-05 mixer 中间状态 = 0;motor_cmd[] = 零推力(per A6 §4.4.2 `Control_Out_Bus` 行;数值 = E4 锁的 disarm PWM 或 normalized 0);
- `Control_Out_Bus.cmd_mask` echo = 0(per A6 §4.4.2 `FMS_Out_Bus.cmd_mask` HARDCODED 全 0,Controller 直 echo);
- `Control_Out_Bus.ctrl_mode` echo = `CMODE_DISARMED`(per A6 §4.4.2 + B2 §4.4.6 锁 enum 数值);
- 各 saturation latch / anti-windup engage flag = false。

#### 4.7.2 DISARMED cmd_mask 行为(C-10 行 §4.2.1)

- D3 在 DISARMED / FS-04 状态下产出 `cmd_mask = 0`(per [D1 §4.1.3 mode-at-reset](../D-fms/D1-fms-functional.md));
- Controller 收到 cmd_mask=0 后:全部 5 环 bypass(§4.4.1);Output Assembler 输出 §4.7.1 安全态(零推力 / disarm PWM / `cmd_mask` echo=0 / `ctrl_mode` echo=disarmed-equiv);
- Controller **不**自行决策"DISARM";仅响应 FMS 给出的 cmd_mask + reset 信号。

#### 4.7.3 Failsafe 行为(per D1 §4.4 触发表)

- D3 在 failsafe 触发后产出新的 `cmd_mask` + `mode` + `failsafe_state`(per D1 FS-01..FS-04 表);Controller 按 §4.2.4 bumpless transfer 切换;无独立 failsafe 决策;
- TR-03(INS validity loss)等关键失效 → D3 拉 `FMS_Out_Bus.reset` + cmd_mask=0(safe state)→ Controller 走 §4.7.1 / §4.7.2 路径;
- **Controller 不消费 `failsafe_state` 字段做控制决策**;仅可在 debug telemetry 中 echo(若 B1 锁定)。

### 4.8 跨引用与下游耦合

| 下游 | E1 提供 |
|---|---|
| [E3 各环算法设计](E3-loops-algorithm.md)(Wave 9)| §4.3 各环输入 / 输出 / anti-windup engage 触发条件 / reset 行为功能层定义,作为 E3 控制律方程注入的功能契约;§4.2.4 bumpless transfer 要求 + §4.4.4 integrator reset 策略 = E3 算法约束 |
| [E4 多旋翼 leaf](E4-multicopter-leaf.md)(Wave 9)| §4.3 (f) 各环 leaf 委托点 + §4.3.5 L-05 mixer 全部内部委托 |
| [E5 性能预算](E5-performance-budget.md)(Wave 10)| §4.1 5 环路清单 + §4.6 输出字段集 = 时间预算分摊单元 |
| [F3 INS 消费规则](../F-ins-contract/F3-consumption-rules.md)(Wave 9)| §4.5 INS 消费列(Controller 部分)+ §4.5 验证-drop 功能意图 — F3 在此意图基础上锁定完整 fallback 表 |
| [D6 FMS↔Controller 接口](../D-fms/D6-fms-controller-interface.md)(Wave 8 sibling)| §4.2.1 cmd_mask 组合表的"激活环路 / 旁路环路 / setpoint 来源"列 — D6 锁位语义,E1 给环路裁剪表;两份共同闭合 FMS↔Controller 接口契约 |
| [D3 Mode Manager](../D-fms/D3-mode-manager.md)(Wave 8 sibling)| §4.7.2 DISARMED 行为约束 + §4.2 cmd_mask 组合覆盖范围 — D3 产出的 cmd_mask 必须落在 §4.2.1 表内或走 §4.2.2 通用规则 |
| [D4 Command Shaper](../D-fms/D4-command-shaper.md)(Wave 8 sibling)| §4.2.1 表"setpoint 来源"列要求 — D4 必须在每个 cmd_mask 组合下填充对应 `FMS_Out_Bus` 字段 |
| [G1 MIL 顶层](../G-harness/G1-mil-toplevel.md)(Wave 10)| §4.6 `Control_Out_Bus` 输出契约 — G1 logsout / scenario 接入点 |
| [H1 验证场景目录](../H-verification/H1-scenario-catalog.md)(Wave 11)| §4.2.1 11 cmd_mask 组合 + §4.7.2 DISARMED + §4.7.3 failsafe = H1 controller-level golden path / edge case 候选场景 |

## 5. 已知风险与悬而未决问题

- **D6 位语义未冻结时 §4.2.1 11 行表的兼容性**
  - 影响:E1 §4.2.1 表 C-01..C-11 假设 D6 §4.3 priority 与 §4.4 mutex 接受的位组合在 Phase 2 多旋翼下覆盖。若 D6 sibling 在 co-seal 评审引入未列组合(例:`POSITION_LOOP + ATTITUDE_LOOP` 同时 active 不 cascade)或锁定某组合非法,§4.2.1 表需修订。
  - 处置:Wave 8 co-seal 评审时核对;变更日志同步;§4.2.2 通用规则已为表外组合提供兜底。

- **`MASK_BIT_ACCELERATION_LOOP=2`(B2 位号 2)Phase 2 不开独立环的决策跨 D6 校验**
  - 影响:E1 §4.1.1 不变量 3 决定 acc 走 L-02 前馈而非独立环;若 D6 §4.x 锁定 acc bit 必须配合独立环,本决策推翻。
  - 处置:Wave 8 co-seal 与 D6 协同;若推翻,E1 §4.1 增补 L-02-acc 子环路或等价;D4 sibling 同步调整 jerk-limited 输出路径。

- **`MASK_BIT_YAW_LOOP=5` / `MASK_BIT_YAW_RATE_LOOP=6` 与主 ATT/RATE 环的子通道融合策略**
  - 影响:§4.1.1 不变量 4 决定 yaw 不形成独立环。若 D6 锁定 yaw absolute 必须独立 cascade(例:某些 FMS mode 下 yaw P 增益与 roll/pitch 不同),L-03 内部需再分子环。
  - 处置:Wave 8 co-seal;E4 leaf 配置如能用增益矩阵分通道实现,则 §4.1.1 不变量 4 保持。

- **`flag.attitude_valid==0` 时的"critical 失效"是 controller-side trigger 还是 FMS-side trigger**
  - 影响:§4.5 / §4.3.4 标 "触发 FMS TR-03",但 Controller 在 INS 失效瞬间是否需要立即输出零推力(controller-side immediate)还是等 D3 下一帧 cmd_mask=0(FMS-side delayed)— 影响响应延迟(20 ms FMS vs 5 ms Controller)。
  - 处置:open;由 F3 + Wave 8 co-seal 联合决策;Phase 2 默认 controller-side immediate 输出零推力 + FMS-side 同步触发 mode 退化。

- **bumpless transfer §4.2.4 路径选择(a vs b)未锁**
  - 影响:cmd_mask 切换时 E1 列两种实现路径(a)新 outer-most 环 integrator 重新初始化使 inner setpoint = 切换前 measurement;(b)D4 sibling 在 setpoint 上做 rate-limited 渐变。两者实现成本与平滑度不同。
  - 处置:由 E3 + D4 Wave 9 协同选择;E1 仅声明功能要求。

- **`Control_Out_Bus.thrust_cmd`(scalar)字段是否存在未由 B1 锁定**
  - 影响:§4.6 输出表中 `thrust_cmd` 标 *(若 B1 锁存在)*;若 B1 后续锁定不存在,L-04 thrust 直接由 L-05 mixer 内部消费,本字段从输出表移除。
  - 处置:open;由 B1 byte-level 锁定 + B4 首跑;变更日志同步。

- **Controller 检测到 illegal cmd_mask(§4.2.3)时是否需要 telemetry 上报**
  - 影响:§4.2.3 标 "Controller_Out_Bus.debug_* 字段"作 illegal_cmd_mask 标记,但具体字段名 / 是否 B1 锁定未确认。若 B1 不留 debug 通道,illegal 检测仅本地 trace,无外部可观测信号 — 增加调试难度。
  - 处置:open;由 E5(Wave 10)决定 debug 字段保留集 + B1 联动。

- **Phase 2 多旋翼 leaf 默认 N=4 quad-X 是否被 E4 sibling 接受**
  - 影响:§4.3.5 L-05 输入 / 输出表标 "Phase 2 多旋翼默认 N=4 quad-X";若 E4 在 Wave 9 决定 Phase 2 同时支持 hex / + / X 多构型,§4.3.5 输入参数集合需扩展。
  - 处置:open;由 E4 锁;E1 仅声明环路存在性,内部细节交 E4。

- **Co-seal batch 内部交叉依赖在 4 份同时 draft 时的不可避免暂态**
  - 影响:本文件 §3 援引 D3 / D4 / D6 sibling,实际它们 status=draft;若 Wave 8 co-seal 评审中任一 sibling 未达 reviewed,E1 复审将一并 hold。
  - 处置:走 RULES §6 第 2 项 "co-seal batch 兄弟可在 draft 状态互相引用"批准的路径;Wave 8 co-seal 一次性锁四份。

## 6. 退出条件复核

对照 [`00-design-plan.md §4.E`](../00-design-plan.md) E1 行:

> **E1 退出条件原文**:启用的环路、cmd_mask 触发的环路裁剪规则、各环职责。**注:cmd_mask 裁剪规则部分需与 D6 同 batch 评审(co-seal),整个 E1 推迟到 Wave 8 启动以避免 D6 → E1 强前置违反**

| # | 退出条件原文(分项)| 本文档依据 | 状态 |
|---|---|---|---|
| 1 | 启用的环路 | §4.1.1 5 环路总表(L-01 Position / L-02 Velocity / L-03 Attitude / L-04 Angular rate / L-05 Mixer-leaf)+ §4.1.1 关键不变量 5 条(全 5 ms 单速率;L-05 多旋翼 leaf;acc 融入 L-02 前馈;yaw 融入 L-03/L-04 子通道;throttle_passthrough 绕开 mixer-内 thrust);§4.1.2 环路 enable 与 cmd_mask 总览 | 满足 |
| 2 | cmd_mask 触发的环路裁剪规则 | §4.2.1 11 行裁剪总表(C-01..C-11 + 通用规则 §4.2.2 + 非法 fallback §4.2.3 + bumpless transfer §4.2.4);**与 D6 sibling co-seal**(§3 依赖表 + co-seal batch 注记);引用 D6 §4.3 priority + D6 §4.4 mutex(per 任务 brief 验收) | 满足(Wave 8 co-seal 前提下 ) |
| 3 | 各环职责 | §4.3.1..§4.3.5 五小节,每环按 (a) 输入 / (b) 输出 / (c) anti-windup engage / (d) validity gating 意图 / (e) reset 行为 / (f) 多旋翼 leaf 委托点 六维度详述;不进入 E3 算法范畴 | 满足 |
| 4 | (附加任务要求)级联 hand-off 语义 | §4.4.1 bypass 时内环 setpoint 来源表 + §4.4.2 bypass 期间外环状态策略(默认 zero-frozen + integrator clear)+ §4.4.3 跨级 anti-windup 信号传播意图 + §4.4.4 cmd_mask 切换时 integrator reset 策略 | 满足 |
| 5 | (附加任务要求)INS_Out_Bus 消费(echo A3 §4.8) | §4.5 echo Controller 列(11 行字段 × 5 环 + gate 位列)+ validity-drop 功能意图(critical / non-critical 区分)+ F3 forward-cite | 满足 |
| 6 | (附加任务要求)输出契约(`Control_Out_Bus` echo B1 §4.4)| §4.6 echo B1 schema(motor_cmd / thrust_cmd / actuator_cmd passthrough / saturation flags / debug / cmd_mask echo / ctrl_mode echo / throttle_cmd echo / reset echo / timestamp) | 满足 |
| 7 | (附加任务要求)reset / disarm / failsafe 行为(per A6) | §4.7.1 init 后稳态(echo A6 §4.2.3)+ §4.7.2 DISARMED cmd_mask 行为 + §4.7.3 failsafe 行为(per D1 §4.4) | 满足 |
| 8 | co-seal note "推迟到 Wave 8 与 D6 同 batch" | §3 依赖表 + Co-seal batch 注记声明 (D3, D4, D6, E1) Wave 8 batch;§5 风险条目"Co-seal batch 内部交叉依赖在 4 份同时 draft 时的不可避免暂态" | 满足 |

## 7. 下游影响

按 [`01-design-relationships.md` §4.5 + §4.6 + §4.8](../01-design-relationships.md) E1 出边:

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [E2 Controller 结构设计](E2-controller-structural.md) | E1 → E2(强前置;Wave 7 已先行 reviewed,Wave 8 co-seal 兼容性核对)| §4.1.1 5 环路清单 + §4.2.1 cmd_mask 裁剪表 = E2 §4.5 各环骨架 + §4.3.2 `Controller_CmdMask_Resolver` 内部规则注入靶子(Wave 8 co-seal 评审兼容性)|
| [E3 各环算法设计](E3-loops-algorithm.md) | E1 → E3(经 E2)| §4.3 各环输入 / 输出 / anti-windup engage 条件 / reset 行为功能层定义 + §4.2.4 bumpless transfer + §4.4.4 reset 策略 |
| [E4 多旋翼 leaf](E4-multicopter-leaf.md) | E1 → E4(经 E3)| §4.3 (f) 各环 leaf 委托点 + §4.3.5 L-05 mixer 全部内部委托 |
| [E5 性能预算](E5-performance-budget.md) | E1 ⇢ E5(弱) | §4.1 5 环路 + §4.6 输出字段 = 时间预算分摊单元 |
| [F3 INS 消费规则](../F-ins-contract/F3-consumption-rules.md) | B5/D1/E1 ⇢ F3 | §4.5 Controller 列 + validity-drop 功能意图 — F3 在此基础上锁完整 fallback 表 |
| [D6 FMS↔Controller 接口](../D-fms/D6-fms-controller-interface.md)(Wave 8 sibling)| ↔ co-seal | §4.2.1 11 行 cmd_mask 组合 → 环路裁剪表;**D6 锁位语义,E1 锁裁剪规则;两份联合闭合 FMS↔Controller 接口** |
| [D3 Mode Manager](../D-fms/D3-mode-manager.md)(Wave 8 sibling)| ⇢ | §4.2.1 表覆盖 D3 产出的 cmd_mask 集合;§4.7.2 DISARMED 行为 |
| [D4 Command Shaper](../D-fms/D4-command-shaper.md)(Wave 8 sibling)| ⇢ | §4.2.1 表 "setpoint 来源" 列 — D4 在每个 cmd_mask 组合下填充对应 `FMS_Out_Bus` setpoint 字段 |
| [H1 验证场景目录](../H-verification/H1-scenario-catalog.md) | D1, E1 ⇢ H1 | §4.2.1 11 cmd_mask 组合 + §4.7.2 DISARMED + §4.7.3 failsafe = H1 controller-side scenario 候选 |

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-08 | controller-architect (E1 author subagent) | 初稿;闭合 E1 退出条件 1–3 + 任务 brief 附加要求 4–7 + co-seal note 第 8 项;`(D3, D4, D6, E1)` Wave 8 co-seal batch 起草态(D3/D4/D6 同步 draft);§4.2.1 11 行 cmd_mask 裁剪总表覆盖 D1 §4.1.1 12 主 mode + 4 failsafe 子 mode 产出的所有典型 cmd_mask 组合 + 通用规则 + 非法 fallback;bypass 默认 zero-frozen + integrator clear;validity-drop critical(attitude)走 controller-side immediate 零推力 + FMS-side 同步触发;FMT-Firmware @ `<pending hash>` 占位 |

## Self-check

- [x] frontmatter 完整,字段值合法
- [x] 上游文档全部存在且 status ≥ reviewed,**或**为本工作项所在 co-seal batch 的同批兄弟(架构 v1 / A3 / A4 / A6 / A7 / B1 / B2 / B3 / C1 / D1 / E2 / F1 全部 reviewed;**D3 / D4 / D6 为 co-seal batch (D3, D4, D6, E1) Wave 8 同批兄弟,§3 依赖表 + Co-seal batch 注记已显式声明,符合 RULES §6 第 2 项**)
- [x] 退出条件逐条复核完成,每条均给出依据(§6 表 8 行覆盖启用环路 / cmd_mask 裁剪规则 / 各环职责 / 级联 hand-off / INS 消费 / 输出契约 / reset / co-seal note)
- [x] 引用路径全部可点击访问(同 design 目录与架构 v1;sibling D3/D4/D6 链接为 Wave 8 co-seal 期间预期路径,与 E2 §3 已建立的引用模式一致)
- [x] 不存在 RULES §5 禁则中的内容(无 `.slx` 截图、无 `.m` 代码、无重复 firmware 实现细节、无重复 bus/enum/param 字段表 — 全部按字段名引用 A3/B1/B2/B3;无 PR/branch 名;依赖关系全部显式列在 §3)
- [N/A] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记 — **本文件 contract_impact=no(功能设计;契约由 B 区拥有),N/A**(per 任务 brief Constraints "no" + RULES §3 第 4 条)
- [N/A] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径 — **本文件不镜像 firmware 契约,仅引用 B 区已镜像的字段名,N/A**(§3 依赖中保留 firmware @ `<pending hash>` 占位以备后续 cmd_mask 位语义首次镜像验证使用)
- [x] 下游影响已沿关系图识别完毕(§7 表覆盖 E2 / E3 / E4 / E5 / F3 / D6 / D3 / D4 / H1 共 9 个下游)
- [x] 文档不超出本工作项范围:不做 E2 结构(只引用)、不做 E3 控制律方程、不做 E4 多旋翼 leaf 内部、不做 E5 性能预算、不做 D6 cmd_mask 位语义(仅消费)、不做 D3/D4 mode 与 shaper 公式、不做 F3 完整 fallback 表(仅功能意图)、不重定义 B 区契约
