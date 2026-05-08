---
work_item: D6
title: FMS↔Controller 接口约定(cmd_mask 位语义 / 参考字段优先级 / 互斥规则)
upstream: [架构 v1 §7.1, 架构 v1 §7.2, 架构 v1 §12.1, 架构 v1 §12.2, A3, A6, B1, B2, D1, D2, E2, D3, D4, E1]
contract_impact: yes
status: draft
authored_at: 2026-05-08
last_reviewed_at:
reviewer_verdict: none
---

# D6 FMS↔Controller 接口约定(cmd_mask 位语义 / 参考字段优先级 / 互斥规则)

## 1. 目的

定义 FMS 通过 `FMS_Out_Bus` 向 Controller 传递的**意图层接口契约**:`cmd_mask` 每一位的语义、当多位同时置位时的参考字段优先级、以及哪些位组合是非法 / Controller 必须拒绝的。本文件是 [D3 Mode Manager](D3-mode-manager.md) / [D4 Command Shaper](D4-command-shaper.md) / [E1 Controller 功能设计](../E-controller/E1-controller-functional.md) 三个 Wave 8 同 batch 兄弟共同遵循的底层契约。

## 2. 范围

**在范围:**
- `FMS_Out_Bus.cmd_mask`(uint32 bitfield,字段类型由 [B1 §4.7](../B-contracts/B1-bus-inventory.md) 锁、位号由 [B2 §4.4.14](../B-contracts/B2-enum-inventory.md) 锁)每一位的**语义定义**:置位 / 清位时 Controller 与 FMS 双方的契约义务
- 多位同时置位时的**参考字段优先级表**(precedence table)
- **互斥规则**:Controller 必须拒绝(或 fall through 到安全态)的非法位组合
- `cmd_mask` 与 `INS_Out_Bus` validity 位 / `INS_Status` 的协同 gate 规则
- D3 / D4 / E1 hand-off 接口形状(只锁列头与契约形状,不填具体值)
- Reset / disarm / failsafe 三态下 `cmd_mask` 的强制取值

**不在范围(由其他工作项处理):**
- `cmd_mask` 字段在 `FMS_Out_Bus` 中的字节偏移与序列化 — 由 [B1 §4.7](../B-contracts/B1-bus-inventory.md) 处理
- bit-position 工作名 + 位号 + 底层整型 — 由 [B2 §4.4.14](../B-contracts/B2-enum-inventory.md) 处理(D6 仅引用,不重定义)
- Stateflow 哪个状态在 `during` / `entry` 上 emit 哪个位组合 — 由 [D3](D3-mode-manager.md) 处理
- 整形器(死区 / rate / jerk / tilt-limit)如何产出 `cmd_mask` 位与对应 setpoint 字段值 — 由 [D4](D4-command-shaper.md) 处理
- Controller 内部位置 / 速度 / 加速度 / 姿态 / 角速度 / yaw / yaw_rate / throttle 各环路的具体启用 / 旁通 / 整形算法 — 由 [E1](../E-controller/E1-controller-functional.md)(本契约的消费侧)处理
- INS 失效情境下 Controller 的具体 fallback 策略(降级 / 保持 / 紧急姿态 hold)— 由 [F3 INS_Out_Bus 消费规则](../F-ins-contract/F3-consumption-rules.md)(Wave 9)处理
- mixer / 分配矩阵层 — 由 [E4 Controller 多旋翼 leaf](../E-controller/E4-multicopter-leaf.md) 处理

## 3. 依赖

> **co-seal batch 声明**:本文件属于 Wave 8 `(D3, D4, D6, E1)` co-seal batch(见 [`01-design-relationships.md` §5 entry 2](../01-design-relationships.md))。本文件与 D3 / D4 / E1 三个**同 batch 兄弟**互相引用,允许在 status=draft 时构成依赖闭环(per [RULES §6 self-check item 2](../RULES.md))。

| 上游 | 引用位置 | 用途 |
|---|---|---|
| [架构 v1 §7.1](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | reviewed | FMS 模块边界:owns 模式管理 / 命令源仲裁 / 命令整形;不进入内环 |
| [架构 v1 §7.2](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | reviewed | Controller 模块边界:消费 `FMS_Out_Bus` + `INS_Out_Bus`,产生 `Control_Out_Bus`,不做模式状态机 |
| [架构 v1 §12.1](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | reviewed | FMS 6 子模块栈;Command Shaper 是 cmd_mask 单一写者 |
| [架构 v1 §12.2](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | reviewed | Controller 内部级联拓扑:Setpoint Decode + 位置 / 速度 / 姿态 / 角速度 / Mixer 七子系统 |
| [A3 模块边界信号清单细化](../A-architecture/A3-module-boundaries.md) | reviewed | §4.4.6.1 `FMS_Out_Bus` setpoint 字段集与 cmd_mask 守卫指针;§4.7 cmd_mask gated 字段总账 |
| [A6 Init/Reset 状态契约](../A-architecture/A6-init-reset-contract.md) | reviewed | §4.4 `cmd_mask` reset 类别 = HARDCODED 全 0(safe state);§4.5 local reset 由 cmd_mask 部分位翻转触发 |
| [B1 Bus 清单](../B-contracts/B1-bus-inventory.md) | reviewed | §4.7.1 `FMS_Out_Bus.cmd_mask` 字段类型 = `uint32` bitfield;§4.9.3 cmd_mask gated 字段对照表;字段 14 在 ledger 中的位置 |
| [B2 Enum 清单](../B-contracts/B2-enum-inventory.md) | reviewed | §4.4.14 `cmd_mask` 位号工作名 + 数值(`MASK_BIT_*`,8 个 Phase 2 位);§4.4.10 `INS_Status`;§4.4.11 `INS_Flag` 位号 |
| [D1 FMS 功能设计](D1-fms-functional.md) | reviewed | §4.6.2 工作位由哪些 mode 在哪些字段产出;cmd_mask reset = 全 0;`Auto_Cmd_Bus.cmd_mask_request` 在 M-11 OFFBOARD 仅"建议",最终采纳由 D6 裁决 |
| [D2 FMS 结构设计](D2-fms-structural.md) | reviewed | §4.6.2 cmd_mask 单一写者 = Command Shaper;Output Assembler 不修改;reset = 全 0 |
| [E2 Controller 结构设计](../E-controller/E2-controller-structural.md) | reviewed | §4.1.1 / §4.4 setpoint decode + mask resolver 路由槽位;cmd_mask 在 §4.1.1 单点解析,不在各环重复解析 |
| [D3 Mode Manager 详细设计](D3-mode-manager.md) | **draft, co-seal batch (D3, D4, D6, E1)** | 消费 D6 §4.6.1 hand-off:每个 mode 在 `during` / `entry` 上 emit 哪些位组合;D6 锁列头与契约形状 |
| [D4 Command Shaper 详细设计](D4-command-shaper.md) | **draft, co-seal batch (D3, D4, D6, E1)** | 消费 D6 §4.6.2 hand-off:每位置位时对应的 setpoint 字段值契约(单位 / 坐标系 / 量化 / 范围) |
| [E1 Controller 功能设计](../E-controller/E1-controller-functional.md) | **draft, co-seal batch (D3, D4, D6, E1)** | 消费 D6 §4.6.3 hand-off:每位置位时启用的环路;清位时旁通 / 推导规则;参考字段优先级 → E1 内部 cascade 拓扑 |
| [F3 INS_Out_Bus 消费规则](../F-ins-contract/F3-consumption-rules.md) | **未启动,Wave 9** | D6 §4.5 给出 INS 失效时 FMS 应 emit 的 cmd_mask 模式;Controller 侧的具体 fallback 由 F3 锁 |

**Firmware 引用(per [RULES §5](../RULES.md) 镜像政策)**:
- `FMT-Firmware/src/model/fms/<vehicle>/lib/FMS_types.h`(`FMS_Out_Bus.cmd_mask` 的 firmware-canonical 字段;commit `FMT-Firmware @ <pending hash>`)
- `FMT-Firmware/src/model/control/<vehicle>/lib/Controller_types.h`(`Control_Out_Bus.cmd_mask` echo;commit `FMT-Firmware @ <pending hash>`)
- `FMT-Firmware/src/model/ins/lib/INS_types.h`(`INS_Out_Bus.INS_Status` / `INS_Flag`;commit `FMT-Firmware @ <pending hash>`)

> 注:`<pending hash>` 占位符与 [A3](../A-architecture/A3-module-boundaries.md) / [A6](../A-architecture/A6-init-reset-contract.md) / [A7](../A-architecture/A7-time-conventions.md) / [B1](../B-contracts/B1-bus-inventory.md) / [B2](../B-contracts/B2-enum-inventory.md) / [B3](../B-contracts/B3-parameter-schema.md) / [B5](../B-contracts/B5-ins-bus-mirror.md) 同期补齐。

## 4. 设计内容

### 4.1 cmd_mask bit position roster(从 B2 §4.4.14 引用)

> **不在 D6 范围**:位名与位号本身。本节**只引用** [B2 §4.4.14](../B-contracts/B2-enum-inventory.md) 已锁定的工作名 + 数值,作为 §4.2 语义定义的列头。任何位名 / 位号变更走 B2 → D6 同步,不在 D6 单方修改。

Phase 2 多旋翼闭环切片必需的 8 位(B2 §4.4.14 已锁,数值标 `(B4 verify)` 直至 contract diff 首跑通过):

| Bit | B2 工作名 | 位号(`(B4 verify)`)| 守卫的 `FMS_Out_Bus` 字段(per [A3 §4.7](../A-architecture/A3-module-boundaries.md) / [B1 §4.9.3](../B-contracts/B1-bus-inventory.md))|
|---:|---|---:|---|
| 0 | `MASK_BIT_POSITION_LOOP` | 0 | `pos_cmd_ned_m[3]` |
| 1 | `MASK_BIT_VELOCITY_LOOP` | 1 | `vel_cmd_ned_mps[3]` |
| 2 | `MASK_BIT_ACCELERATION_LOOP` | 2 | `acc_cmd_ned_mps2[3]` |
| 3 | `MASK_BIT_ATTITUDE_LOOP` | 3 | `att_cmd_quat[4]` 与 / 或 `att_cmd_euler_rad[3]` |
| 4 | `MASK_BIT_RATE_LOOP` | 4 | `ang_rate_cmd_b_radps[3]` |
| 5 | `MASK_BIT_YAW_LOOP` | 5 | `yaw_cmd_rad` |
| 6 | `MASK_BIT_YAW_RATE_LOOP` | 6 | `yaw_rate_cmd_radps` |
| 7 | `MASK_BIT_THROTTLE_PASSTHROUGH` | 7 | `throttle_cmd` |

底层整型 `uint32`(B2 §4.4.14 锁定;架构 v1 §5.1.5 `cmd_mask` 上限 32 位)。Bits 8..31 = reserved(future Phase 3+ 横向 / 纵向通道分离 / 直接力 / 直接力矩 / 浮空滑行模式等;D6 不预留具体语义)。

### 4.2 Per-bit semantic definition(逐位语义)

每位的语义按以下统一格式定义:**置位含义** / **清位含义** / **validity 依赖** / **reset-time 取值** / **生产者** / **消费者**。

#### 4.2.1 `MASK_BIT_POSITION_LOOP`(bit 0)

| 项 | 内容 |
|---|---|
| **置位含义(SET = 1)** | `FMS_Out_Bus.pos_cmd_ned_m[3]` 字段**有效**,Controller 必须把它作为**最外环位置环**的参考 |
| **清位含义(CLEAR = 0)** | `pos_cmd_ned_m[3]` 字段值**未定义**(per A6 §4.4.2 reset 时 = ZERO,但运行期清位时 D4 不保证 ZERO);Controller **必须不**消费该字段;位置环旁通,内层环路从下一个置位的外环或 INS 当前位置出发 |
| **Validity 依赖** | `INS_Out_Bus.INS_Status >= INS_STATUS_READY` **且** `INS_Flag.INS_FLAG_BIT_POSITION_VALID = 1`(per [B2 §4.4.10 / §4.4.11](../B-contracts/B2-enum-inventory.md));若 validity 未满足,置位本身 = **无效请求**,FMS **不应** emit(per §4.5),Controller fallback per F3 |
| **Reset-time 取值** | 0(per A6 §4.4.2 cmd_mask = HARDCODED 全 0)|
| **生产者** | FMS Command Shaper(D2 §4.6.2)|
| **消费者** | Controller Setpoint Decode + Mask Resolver(E2 §4.1.1)→ 位置环(E2 §4.1.2)|

#### 4.2.2 `MASK_BIT_VELOCITY_LOOP`(bit 1)

| 项 | 内容 |
|---|---|
| **置位含义** | `vel_cmd_ned_mps[3]` 有效;Controller 把它作为**速度环参考**(可作 outer-most 速度环,或在 BIT_POS 同时置位时作位置环输出的 feed-forward / 替代内层参考,per §4.3 优先级表) |
| **清位含义** | `vel_cmd_ned_mps[3]` 未定义;Controller 不消费该字段;速度环参考由 BIT_POS 输出级联推导,或速度环旁通 |
| **Validity 依赖** | `INS_Status >= READY` **且** `INS_FLAG_BIT_VELOCITY_VALID = 1`;若仅 BIT_VEL 单独置位(无 BIT_POS),则**不**强制 `INS_FLAG_BIT_POSITION_VALID = 1` |
| **Reset-time 取值** | 0 |
| **生产者** | FMS Command Shaper |
| **消费者** | Controller 速度环(E2 §4.1.3)|

#### 4.2.3 `MASK_BIT_ACCELERATION_LOOP`(bit 2)

| 项 | 内容 |
|---|---|
| **置位含义** | `acc_cmd_ned_mps2[3]` 有效;Controller 把它作为**加速度参考**(典型用作速度环的 feed-forward,或在某些 thrust-vector 模式下作直接加速度命令)|
| **清位含义** | `acc_cmd_ned_mps2[3]` 未定义;不消费 |
| **Validity 依赖** | 与 BIT_VEL 一致(`INS_FLAG_BIT_VELOCITY_VALID = 1`,加速度通常由速度环路使用;独立 acc 模式由 E1 决定是否需要 attitude validity)|
| **Reset-time 取值** | 0 |
| **生产者** | FMS Command Shaper |
| **消费者** | Controller 速度环 feed-forward 通道(E1 锁具体路径)|

#### 4.2.4 `MASK_BIT_ATTITUDE_LOOP`(bit 3)

| 项 | 内容 |
|---|---|
| **置位含义** | 姿态参考有效。**字段二选一**:`att_cmd_quat[4]`(优先,per §4.3.4)**或** `att_cmd_euler_rad[3]`(legacy 兼容);D4 必须只填一个非零字段,另一个保持 ZERO(per A6 §4.4.2)|
| **清位含义** | 两个姿态字段都未定义;Controller 不消费;姿态环参考由 BIT_VEL / BIT_POS 级联(thrust-vector 推导 desired tilt + yaw)推导,或姿态环旁通(BIT_RATE 直接驱动)|
| **Validity 依赖** | `INS_FLAG_BIT_ATTITUDE_VALID = 1`(quat)+ `INS_FLAG_BIT_HEADING_VALID = 1`(欧拉 yaw 分量,若用 euler)|
| **Reset-time 取值** | 0;`att_cmd_quat` reset = identity `(1,0,0,0)`(per A6 §4.4.2 + A2 §4.10);`att_cmd_euler_rad` reset = ZERO |
| **生产者** | FMS Command Shaper |
| **消费者** | Controller 姿态环(E2 §4.1.4)|

> **att_quat vs att_euler 二选一裁决**:见 §4.3.4。D6 锁定:**同帧内两位都置位 = 非法**(实际只有 1 个 BIT_ATTITUDE_LOOP,但 D4 在两个字段间二选一)。

#### 4.2.5 `MASK_BIT_RATE_LOOP`(bit 4)

| 项 | 内容 |
|---|---|
| **置位含义** | `ang_rate_cmd_b_radps[3]` 有效;Controller 把它作为**角速度环参考**(body-frame);如果 BIT_ATTITUDE_LOOP 同时清位,则角速度环 = outer-most attitude-domain 环路(直接率控,典型 ACRO mode)|
| **清位含义** | `ang_rate_cmd_b_radps[3]` 未定义;角速度环参考由姿态环输出推导(典型 cascade 行为)|
| **Validity 依赖** | `INS_FLAG_BIT_ATTITUDE_VALID = 1`(IMU 给的 body-rate 测量 + attitude 解算;若 attitude invalid 则陀螺 raw 是否可独立用于 rate loop 由 F3 决定;D6 默认要求 attitude valid)|
| **Reset-time 取值** | 0 |
| **生产者** | FMS Command Shaper |
| **消费者** | Controller 角速度环(E2 §4.1.5)|

#### 4.2.6 `MASK_BIT_YAW_LOOP`(bit 5)

| 项 | 内容 |
|---|---|
| **置位含义** | `yaw_cmd_rad` 有效;Controller 把它作为**偏航角参考**;通常与 BIT_ATTITUDE_LOOP **互补**(BIT_ATT 给的姿态参考是 roll/pitch;yaw 走独立 BIT_YAW)或与 quat 内嵌冗余(per §4.3.5)|
| **清位含义** | `yaw_cmd_rad` 未定义;偏航参考由 BIT_ATT_QUAT 内嵌的 yaw 提供,或由 BIT_YAW_RATE 积分推导,或姿态环 yaw 通道旁通(rate-only)|
| **Validity 依赖** | `INS_FLAG_BIT_HEADING_VALID = 1` |
| **Reset-time 取值** | 0 |
| **生产者** | FMS Command Shaper |
| **消费者** | Controller 姿态环 yaw 通道 |

#### 4.2.7 `MASK_BIT_YAW_RATE_LOOP`(bit 6)

| 项 | 内容 |
|---|---|
| **置位含义** | `yaw_rate_cmd_radps` 有效;Controller 把它作为**偏航角速度参考**;典型用于 manual mode pilot stick → yaw_rate(经 D4 整形 + 死区 + expo)|
| **清位含义** | `yaw_rate_cmd_radps` 未定义;偏航角速度由 BIT_YAW 角度环输出推导,或角速度环 yaw 通道旁通 |
| **Validity 依赖** | `INS_FLAG_BIT_HEADING_VALID = 1`(若 yaw_rate 与 yaw 闭环联合),或仅 `INS_FLAG_BIT_ATTITUDE_VALID = 1`(若纯 rate 模式;F3 锁)|
| **Reset-time 取值** | 0 |
| **生产者** | FMS Command Shaper |
| **消费者** | Controller 角速度环 yaw 通道 |

#### 4.2.8 `MASK_BIT_THROTTLE_PASSTHROUGH`(bit 7)

| 项 | 内容 |
|---|---|
| **置位含义** | `throttle_cmd`(范围 0..1)**直传**给 Controller mixer / 总推力分配,**绕过**位置 / 速度 / 加速度环对总推力的闭环计算;典型用于 MANUAL / STABILIZE / ACRO 模式下 pilot collective 直传 |
| **清位含义** | `throttle_cmd` 未定义;**总推力**由 Controller 内部位置 / 速度 / 加速度环计算(典型 thrust-vector 推导);Controller **不**消费 `FMS_Out_Bus.throttle_cmd` 字段值 |
| **Validity 依赖** | 无(throttle passthrough 是开环量;不依赖任何 INS flag)|
| **Reset-time 取值** | 0 |
| **生产者** | FMS Command Shaper |
| **消费者** | Controller mixer / 总推力分配通道(E2 §4.1.6 / E4)|

> **重要**:BIT_THROTTLE_PASSTHROUGH 与 BIT_POS / BIT_VEL / BIT_ACC **互斥**(参 §4.4 互斥规则)— 总推力来源必须**单一**(闭环推导 OR 直传,二选一)。

### 4.3 Reference field priority(优先级表)

当多位同时置位时,Controller 按**外→内、绝对→相对、复合→分量**三原则解析参考。下表逐对给出契约语义。

#### 4.3.1 总则(Three Rules)

- **R-Pri-1(外环优先)**:已置位的最外环参考是参考链的根;其下层环参考(若同时置位)解释为 feed-forward 或被优先参考的级联输出**覆盖**(由 E1 锁实际算法)。
- **R-Pri-2(complex 优先)**:同环位置上,复合参考(quat 含 yaw)优先于分量参考(独立 yaw),除非 D4 显式将 yaw 槽置位以表达"quat 不含 yaw,yaw 走独立通道"。
- **R-Pri-3(passthrough 互斥优先)**:THROTTLE_PASSTHROUGH 与位置 / 速度 / 加速度链是**模式互斥**,不允许"部分 passthrough"。

#### 4.3.2 平移域(translational)优先级:POS / VEL / ACC

| BIT_POS | BIT_VEL | BIT_ACC | 语义裁决 | 典型 mode |
|:---:|:---:|:---:|---|---|
| 1 | 0 | 0 | 位置环 outer-most;速度 / 加速度参考 = 位置环输出推导 | M-08 LOITER / M-10 RTL hold-frame |
| 1 | 1 | 0 | 位置环 outer-most;`vel_cmd` = 速度环 **feed-forward**(典型 trajectory-tracking);如位置环输出与 `vel_cmd` 显著不一致,以位置环输出为主参考(E1 锁混合权)| M-09 MISSION trajectory |
| 1 | 1 | 1 | 位置环 outer-most;`vel_cmd` = 速度环 FF;`acc_cmd` = 加速度 FF;典型多项式轨迹三阶 FF | M-09 MISSION 轨迹精跟 / M-11 OFFBOARD trajectory |
| 0 | 1 | 0 | 速度环 outer-most(位置环旁通);`vel_cmd` 直接驱动 | M-04 ALTITUDE_HOLD(z-vel)/ M-07 POS_CTRL(stick→velocity)|
| 0 | 1 | 1 | 速度环 outer-most;`acc_cmd` = 加速度 FF | M-11 OFFBOARD vel+acc |
| 0 | 0 | 1 | 加速度直传到速度环 / thrust-vector 推导;**罕见**;E1 锁是否合法(默认认为合法但产生姿态参考必须由 BIT_ATT 或 quat-from-acc 推导)| M-11 OFFBOARD acc-only(实验)|
| 1 | 0 | 1 | 位置环 outer-most;速度 FF 缺失但加速度 FF 存在 — **D6 锁:合法但 E1 必须告警**(典型 D4 错配,日志 / debug 级)| 异常 / 调试 |
| 0 | 0 | 0 | 平移域**全部**旁通;Controller 不闭合平移环;总推力来源必须由 BIT_THROTTLE_PASSTHROUGH 提供(否则触发互斥规则,见 §4.4)| M-02 MANUAL / M-03 STABILIZE / M-12 ACRO |

#### 4.3.3 姿态域(rotational)优先级:ATT / RATE

| BIT_ATT | BIT_RATE | 语义裁决 | 典型 mode |
|:---:|:---:|---|---|
| 1 | 0 | 姿态环 outer-most;角速度参考由姿态环输出推导 | M-03 STABILIZE / M-04..M-11 闭环间接 |
| 1 | 1 | 姿态环 outer-most;`ang_rate_cmd` = 角速度环 **feed-forward**;典型 trajectory + 高速跟随 | M-09 MISSION 高动态 |
| 0 | 1 | 角速度环 outer-most(姿态环旁通);典型 ACRO / MANUAL rate stick 直传 | M-02 MANUAL / M-12 ACRO |
| 0 | 0 | 姿态域**全部**旁通;Controller 不闭合姿态环 — 仅在 BIT_THROTTLE_PASSTHROUGH + 全平移旁通的"开环演示模式"下出现;否则触发互斥规则(见 §4.4)| 罕见;调试 |

#### 4.3.4 ATT_QUAT vs ATT_EUL 字段二选一

D6 锁定:`MASK_BIT_ATTITUDE_LOOP` 是**单一位**,姿态字段在 D4 端二选一(不在 mask 端二选一)。规则:

| `att_cmd_quat` | `att_cmd_euler_rad` | 裁决 |
|:---:|:---:|---|
| ≠ identity | = ZERO | **合法**;Controller 消费 quat;**首选** |
| = identity | ≠ ZERO | **合法(legacy)**;Controller 消费 euler;Controller 内部 euler→quat 转换 per A2 §4.10 ZYX 顺序;**E1 必须告警**(legacy path)|
| ≠ identity | ≠ ZERO | **非法**(D4 错配 — 两字段都填了非默认值);Controller **必须**优先消费 quat,丢弃 euler,且发 `ErrorCode` `ERR_FMS_CMD_INCONSISTENT`(per [B2 §4.4.12 ErrorCode](../B-contracts/B2-enum-inventory.md);具体名留 B2 / E1 锁)|
| = identity | = ZERO | **合法**(D6 视为 BIT_ATTITUDE_LOOP 下"姿态参考 = 中性");Controller 把它当成 hover-level 参考 |

> 注:`identity` = `(w=1, x=0, y=0, z=0)`;`ZERO` = 三个 euler 分量全 0。

#### 4.3.5 YAW vs YAW_RATE vs ATT(yaw 通道三态)

姿态参考的 yaw 通道有三个潜在来源:`att_cmd_quat` 内嵌的 yaw、独立的 `yaw_cmd_rad`(BIT_YAW)、以及 `yaw_rate_cmd_radps`(BIT_YAW_RATE)。优先级:

| BIT_ATT (quat) | BIT_YAW | BIT_YAW_RATE | 裁决(yaw 通道) |
|:---:|:---:|:---:|---|
| 1 (quat) | 0 | 0 | yaw = quat 内嵌 yaw |
| 1 (quat) | 1 | 0 | **非法**(双重定义角度);Controller 优先 BIT_YAW(显式高于隐式),发 `ERR_FMS_CMD_INCONSISTENT` 警告 |
| 1 (quat) | 0 | 1 | yaw 角 = quat 内嵌 yaw(目标);yaw 角速度 = `yaw_rate_cmd` 作为 yaw 环 FF;**合法且常用** |
| 1 (quat) | 1 | 1 | **非法**;同上 — 优先 BIT_YAW + BIT_YAW_RATE 显式对(忽略 quat 内嵌 yaw),发警告 |
| 1 (euler) | 0 | 0 | yaw = euler.yaw |
| 1 (euler) | 1 | 0 | **非法**;同 quat-冲突;优先 BIT_YAW |
| 1 (euler) | 0 | 1 | yaw 角 = euler.yaw;yaw_rate 作 FF — 合法 |
| 0 (BIT_ATT 清位)| 1 | 0 | yaw 由 BIT_YAW 独立环驱动;Controller 内部对应 yaw-only attitude 环 |
| 0 | 0 | 1 | yaw_rate 直传到 yaw 角速度环;典型 MANUAL / ACRO yaw-stick |
| 0 | 1 | 1 | yaw 角环 outer-most;yaw_rate 是 yaw 环 FF — 合法 |
| 0 | 0 | 0 | yaw 通道旁通;由 BIT_RATE 的 z-rate 分量驱动(若置位);否则 yaw 通道无源(Controller 进入安全 hold) |

> **冲突仲裁原则**:**显式优于隐式**(BIT_YAW > quat 内嵌 yaw);**速率优于积分**(BIT_YAW_RATE 直接覆盖 yaw 环输出,如 stick 释放则角度命令重新生效)。

#### 4.3.6 全表(precedence summary)

外→内的总优先级(只列**置位**的环路):

```
THROTTLE_PASSTHROUGH ─┬─ (跳过平移域闭环,直接 mixer)
                      │
POS ─→ VEL ─→ ACC ─→ thrust-vector ─→ ATT(roll/pitch)─→ RATE ─→ mixer
                                              │
                                              ├─ YAW(yaw 角)─→ YAW_RATE ─→ rate.z 通道 ─→ mixer
                                              └─ (quat 内嵌 yaw,若 BIT_YAW 清位)
```

E1 在此契约下决定每环的具体跨越路径(直传 vs cascaded vs feed-forward)。

### 4.4 Mutual exclusion rules(互斥与非法组合)

Controller **必须**对以下组合做出确定性响应:**reject + 触发安全 fallback** 或**warning + 仲裁**。下表 `Action` 列锁定 Controller 行为。

| # | 非法 / 仲裁组合 | 原因 | Controller Action |
|---:|---|---|---|
| MX-1 | `cmd_mask = 0`(全 0)| 无任何环路启用;无总推力来源;但**reset/disarmed 安全态**(per A6 §4.4.2)| **合法但解释为 disarmed-equivalent**:Controller 输出零推力 + 角速度环零参考 + integrator 清零(per A6 §4.5.2 local reset)。**不**触发错误码;这是契约定义的安全姿态 |
| MX-2 | BIT_THROTTLE_PASSTHROUGH=1 **且**(BIT_POS=1 或 BIT_VEL=1 或 BIT_ACC=1)| 总推力来源冲突:闭环推导 vs 直传,二选一 | **拒绝**:Controller 优先 THROTTLE_PASSTHROUGH(显式 > 隐式);忽略 POS / VEL / ACC 的总推力分量,仅用其方向分量(若 ATT 清位则忽略整个平移域);发 `ERR_FMS_CMD_INCONSISTENT` 警告。**E1 实现**:位置 / 速度 / 加速度环旁通,只走姿态 / mixer 链 |
| MX-3 | BIT_POS=1 **但** `INS_FLAG_BIT_POSITION_VALID=0` | 位置参考无可靠位置反馈 | **拒绝**:Controller 把 BIT_POS 视为清位,降级到 BIT_VEL / BIT_ATT;并发 `ERR_INS_INVALID` 警告。FMS 层 §4.5 规定 FMS **不应** emit BIT_POS 当 INS position invalid;若依然 emit,Controller 必须 fallback |
| MX-4 | BIT_VEL=1 **但** `INS_FLAG_BIT_VELOCITY_VALID=0` | 同 MX-3 | 同 MX-3:降级到 BIT_ATT;发 `ERR_INS_INVALID` |
| MX-5 | BIT_ATT=1 **但** `INS_FLAG_BIT_ATTITUDE_VALID=0` | 姿态参考无可靠姿态反馈 | **危险**:Controller 进入 emergency stabilize(细节 F3 锁);D6 锁:Controller 必须**保持上一帧** ATT 或进入降级 attitude hold(由 F3 决定);发 `ERR_INS_INVALID` 严重级 |
| MX-6 | BIT_RATE=1 **但** `INS_FLAG_BIT_ATTITUDE_VALID=0` | 角速度环依赖 IMU + attitude solution | 同 MX-5;F3 决定是否允许 raw-gyro-only 应急路径 |
| MX-7 | BIT_ATT=0 **且** BIT_RATE=0 **且** BIT_THROTTLE_PASSTHROUGH=1 | 仅总推力直传,无任何姿态稳定 | **合法但极特殊**:仅在 ground-test / motor-test variant 出现;Phase 2 多旋翼 MIL **不应**在飞行场景出现;**E1 实现**:Controller 输出 throttle 直传 + 全部 attitude/rate 输出 = 0;发 INFO 级标记。Phase 2 不在 D3 任何 mode 中出现此组合 |
| MX-8 | BIT_THROTTLE_PASSTHROUGH=0 **且**(BIT_POS=0 **且** BIT_VEL=0 **且** BIT_ACC=0)**且**(BIT_ATT=1 或 BIT_RATE=1)| 姿态 / 角速度有参考但无总推力源 | **非法**:Controller 总推力 = 0(零推力姿态参考无意义)+ 发 `ERR_FMS_CMD_INCONSISTENT`;**或** Controller fallback 到 hover-throttle 默认值(由 E1 锁;D6 默认要求 zero-throttle)|
| MX-9 | 同帧 quat ≠ identity **且** euler ≠ ZERO(per §4.3.4)| 姿态字段双填 | 优先 quat,丢弃 euler,发 `ERR_FMS_CMD_INCONSISTENT` |
| MX-10 | BIT_ATT=1 **且** BIT_YAW=1 (重叠 yaw)| yaw 通道双重定义 | 优先 BIT_YAW(per §4.3.5);发 INFO/WARNING(D6 锁:WARNING) |

#### 4.4.1 Truth-table(Phase 2 8 位组合 — 仅列合法 + 关键非法,per §4.3 + §4.4)

> 256 = 2^8 全组合不便枚举;下表只列**合法核心**(对应 D1 §4.4 12 主模式)+ **关键非法触发**。完整位级合法性由 D3 hand-off 表填(见 §4.6.1)。

| 组合(POS,VEL,ACC,ATT,RATE,YAW,YAWR,THR)| 合法 / 非法 | 对应 mode 类(per D1)| 备注 |
|---|---|---|---|
| `(0,0,0,0,0,0,0,0)` | 合法 | reset / disarmed | A6 §4.4.2 + MX-1 |
| `(0,0,0,0,1,0,1,1)` | 合法 | M-02 MANUAL / M-12 ACRO | rate + yaw_rate + throttle direct |
| `(0,0,0,1,0,0,1,1)` | 合法 | M-03 STABILIZE | attitude + yaw_rate + throttle direct |
| `(0,1,0,1,0,0,1,0)` | 合法 | M-04 ALTITUDE_HOLD | vel(z)+ att + yaw_rate;throttle 闭环 |
| `(0,1,0,1,0,1,0,0)` | 合法 | M-07 POS_CTRL(stick→vel)| vel + att + yaw 角 |
| `(1,0,0,1,0,1,0,0)` | 合法 | M-05 LOITER / M-08 hold | pos + att + yaw |
| `(1,1,0,1,0,1,0,0)` | 合法 | M-09 MISSION 一阶 FF | pos + vel FF + att + yaw |
| `(1,1,1,1,1,1,0,0)` | 合法 | M-09 MISSION 三阶 FF | 全 FF |
| `(0,0,0,1,0,1,0,0)` | 合法 | M-06 RTL approach | att + yaw |
| `(1,0,0,0,0,0,0,1)` | **非法 MX-2** | — | THR + POS 互斥 |
| `(0,0,0,0,0,0,0,1)` | 合法但需 Phase | MX-7 | ground-test only |
| `(1,0,0,0,1,0,0,0)` | **非法 MX-8** | — | rate 有参考但无总推力源 |
| `(0,0,0,1,0,1,1,0)` | 合法 | — | attitude + yaw + yaw_rate FF(yaw 闭环 + FF) |

> 余下组合:**默认合法**当且仅当(a)平移 / 姿态 / yaw 三域分别满足 §4.3 优先级表的某行;(b)总推力源单一(THR 直传 OR 闭环推导,不重叠);(c)所有置位的 BIT_* 通过 §4.5 INS validity gate。

### 4.5 Validity / failsafe interaction(INS 失效情境下的 cmd_mask 政策)

D6 锁定 FMS 在 INS 不同失效组合下**应当** emit 的 cmd_mask 模式(下游 Controller 行为由 [F3](../F-ins-contract/F3-consumption-rules.md) 锁)。

| 失效情境 | `INS_Status` / `INS_Flag` | FMS 应 emit 的 cmd_mask 政策 |
|---|---|---|
| INS 完全失效 | `INS_Status < INS_STATUS_READY` | **cmd_mask = 0**(全部清位);Mode Manager 切到 `FAIL_INS_INVALID` 失效子模式(per D1 §4.4 + B2 §4.4.7);Controller 见全 0 mask = disarmed-equivalent(per MX-1)。**强制**(per A6 §4.5.3)|
| INS 位置失效 | `INS_FLAG_BIT_POSITION_VALID=0` 但 attitude / velocity / heading 有效 | FMS **不得** emit BIT_POS;允许 BIT_VEL / BIT_ATT / BIT_YAW / BIT_YAW_RATE / BIT_THROTTLE_PASSTHROUGH;Mode Manager 降级 M-08/M-09/M-10 到 M-04 (alt hold) 或 M-03 STABILIZE |
| INS 速度失效 | `INS_FLAG_BIT_VELOCITY_VALID=0` 但 attitude / heading 有效 | FMS **不得** emit BIT_POS / BIT_VEL / BIT_ACC;允许 BIT_ATT / BIT_RATE / BIT_YAW / BIT_YAW_RATE / BIT_THROTTLE_PASSTHROUGH;Mode Manager 降级到 M-03 STABILIZE |
| INS 姿态失效 | `INS_FLAG_BIT_ATTITUDE_VALID=0` | **危险**:FMS 触发 emergency;`cmd_mask = 0`(per A6 §4.5.3);Mode Manager 切到 `FAIL_INS_INVALID`;Controller 进入紧急姿态 hold(F3 锁)|
| INS 偏航失效 | `INS_FLAG_BIT_HEADING_VALID=0` 但其他 OK | FMS **不得** emit BIT_YAW(角度环);可继续 emit BIT_YAW_RATE(纯 rate);允许 BIT_ATT(roll/pitch only)BIT_RATE 等 |
| GPS 失效(单独)| `INS_FLAG_BIT_GPS_VALID=0`,但 INS 整体仍 ready | FMS 视具体情境降级:水平位置可能依旧有效(光流 / 视觉),由 INS_Flag 仲裁,而非直接由 GPS_VALID gate;D1 §4.5 已锁 |

> **此表锁定 FMS 应有的"良民"行为**;Controller 在见到违反此表的 cmd_mask + INS validity 组合时按 §4.4 互斥规则处理(MX-3..MX-6)。F3 进一步锁 Controller 的具体降级算法(emergency hold / hold-last / damp-and-descend)。

### 4.6 D3 / D4 / E1 hand-off 表(列头与契约形状)

D6 锁列头与契约形状,具体值由各 sibling 在自己文档内填。

#### 4.6.1 D3 hand-off — Mode → cmd_mask emission table

D3 内部 Stateflow 状态(per D1 §4.4 共 12 主模式 + 4 失效子模式)在 `during` / `entry` 上 emit 的 cmd_mask 位组合,由 D3 在自己文档的 mode-cmd 映射表中填。D6 锁定列头:

| Mode 工作名 | `MASK_BIT_POSITION_LOOP` | `MASK_BIT_VELOCITY_LOOP` | `MASK_BIT_ACCELERATION_LOOP` | `MASK_BIT_ATTITUDE_LOOP` | `MASK_BIT_RATE_LOOP` | `MASK_BIT_YAW_LOOP` | `MASK_BIT_YAW_RATE_LOOP` | `MASK_BIT_THROTTLE_PASSTHROUGH` | 备注 |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|---|
| (D3 fills, e.g., M-02 MANUAL)| 0 | 0 | 0 | 0 | 1 | 0 | 1 | 1 | 例:`(0,0,0,0,1,0,1,1)`,per §4.4.1 |

**契约形状**:
- 每行所选位组合**必须**在 §4.4.1 truth-table 的"合法"集合中(D6 enforces);D3 不得 emit `非法 MX-*` 组合。
- `Auto_Cmd_Bus.cmd_mask_request` 在 M-11 OFFBOARD 中由 D3 仲裁后采纳(per D1 §4.6.2 + §4.5.5);D6 锁:**采纳前**必须经 §4.4 互斥过滤。
- 单一写者:Stateflow 在 `during` 上写、Output Assembler 不修改(per D2 §4.6.2)。
- Reset 时 = 全 0(per A6 §4.4.2)。

#### 4.6.2 D4 hand-off — Shaper → cmd_mask production + setpoint 字段值契约

D4 在每个 mode 下产出的 setpoint 字段值与 cmd_mask 位的**配对契约**:

| 位 | 对应字段 | 字段值产出契约(D4 fills numerics)| 单位 | 坐标系 | Reset 值 |
|---|---|---|---|---|---|
| `MASK_BIT_POSITION_LOOP` | `pos_cmd_ned_m[3]` | 3-vector,经 mode-conditional shape(e.g., M-08 hold = INS 当前 pos 锁存;M-09 = 任务 wp 插值)| `_m`(per A2)| NED | `(0,0,0)`(A6 §4.4.2)|
| `MASK_BIT_VELOCITY_LOOP` | `vel_cmd_ned_mps[3]` | 3-vector;经 stick→vel 整形 + rate-limit(D4)| `_mps` | NED | `(0,0,0)` |
| `MASK_BIT_ACCELERATION_LOOP` | `acc_cmd_ned_mps2[3]` | 3-vector;经 jerk-limited shape | `_mps2` | NED | `(0,0,0)` |
| `MASK_BIT_ATTITUDE_LOOP` | `att_cmd_quat[4]` 或 `att_cmd_euler_rad[3]` | quat 优先;经 tilt-limit + 模式 conditional shape;**二选一,另一字段 = ZERO/identity** | 无单位 / `_rad` | NED→body | quat=identity;euler=ZERO |
| `MASK_BIT_RATE_LOOP` | `ang_rate_cmd_b_radps[3]` | 3-vector body-frame;经 rate-stick + expo + dead-band | `_radps` | body | `(0,0,0)` |
| `MASK_BIT_YAW_LOOP` | `yaw_cmd_rad` | scalar,经 yaw-target shape | `_rad` | NED→body | `0` |
| `MASK_BIT_YAW_RATE_LOOP` | `yaw_rate_cmd_radps` | scalar,经 yaw-rate stick + dead-band | `_radps` | NED→body(z 分量)| `0` |
| `MASK_BIT_THROTTLE_PASSTHROUGH` | `throttle_cmd` | scalar,范围 0..1,经 throttle-curve shape(per D4 + D5 leaf)| `_n01` | N/A | `0` |

**契约形状**:
- D4 必须保证:**位置位与字段值的存在性同步**(BIT_X = 1 ⇔ 字段值是经过 shape 的、有意义的;BIT_X = 0 ⇔ 字段值 = 该字段 reset-time 默认值,per A6 §4.4.2)。
- 单位 / 坐标系 / 量化必须严格遵循 [A2 §4.10](../A-architecture/A2-naming-conventions.md) + [B1 §4.7.1](../B-contracts/B1-bus-inventory.md) 已锁的字段 schema。
- D4 在 ATT 通道只能填 quat **或** euler 中**一个**(per §4.3.4 + MX-9)。
- D4 不得在同帧产生 §4.4 列出的非法组合;若 mode 期望某非法组合,D3 的 Stateflow 必须在该 mode 入口前先调整 cmd_mask 位组合。

#### 4.6.3 E1 hand-off — cmd_mask → loop trim rules

E1 据 §4.2 / §4.3 / §4.4 锁定每环的启用 / 旁通 / cascade 决策。D6 锁列头:

| `cmd_mask` 位 | 当此位 = 1 | 当此位 = 0 | E1 fills algorithm details |
|---|---|---|---|
| `MASK_BIT_POSITION_LOOP` | 启用位置环;参考 = `pos_cmd_ned_m`;输出 → 速度环参考(若 BIT_VEL=0)或 速度环 FF(若 BIT_VEL=1)| 位置环旁通;速度环参考 = `vel_cmd_ned_mps`(若 BIT_VEL=1)或 速度环旁通 | E1 §4.x 位置环 anti-windup / 限幅 / cascade |
| `MASK_BIT_VELOCITY_LOOP` | 速度环参考 = `vel_cmd_ned_mps`(混合规则 per §4.3.2)| 速度环参考 = 位置环输出推导(若 BIT_POS=1)或 旁通 | E1 §4.x 速度环 |
| `MASK_BIT_ACCELERATION_LOOP` | 加速度作为速度环 FF | 加速度由速度环输出推导 | E1 §4.x acceleration FF |
| `MASK_BIT_ATTITUDE_LOOP` | 启用姿态环;参考 quat/euler(per §4.3.4)| 姿态参考 = 平移环 thrust-vector 推导(若任一平移位置位)或 姿态环旁通 | E1 §4.x 姿态环 |
| `MASK_BIT_RATE_LOOP` | 角速度参考 = `ang_rate_cmd_b_radps`(混合规则)| 角速度参考 = 姿态环输出推导(若 BIT_ATT=1)| E1 §4.x 角速度环 |
| `MASK_BIT_YAW_LOOP` | yaw 角参考 = `yaw_cmd_rad`(覆盖 quat 内嵌 yaw,per §4.3.5)| yaw 角参考 = quat 内嵌(若 BIT_ATT=1)或 yaw 角旁通 | E1 §4.x yaw 通道 |
| `MASK_BIT_YAW_RATE_LOOP` | yaw rate 参考 = `yaw_rate_cmd_radps`(混合规则)| yaw rate 参考 = yaw 角环输出推导 | E1 §4.x yaw rate 通道 |
| `MASK_BIT_THROTTLE_PASSTHROUGH` | 总推力 = `throttle_cmd` 直传(绕过 Z 速度 / 加速度环)| 总推力 = Z 平移环输出推导(thrust-vector)| E1 §4.x 总推力来源仲裁 |

**契约形状**:
- E1 必须保证:**位置位与环路启用一一对应**;清位时**不得**消费对应字段。
- E1 必须保证:**优先级表(§4.3)** 在 cascade 拓扑实现中体现;违反需 D6 review。
- E1 必须保证:**互斥规则(§4.4 MX-1..MX-10)** 在内部由 Mask Resolver(per E2 §4.1.1)在每帧入口检测;触发时按 Action 列规定行为。
- E1 必须保证:**INS validity gate(§4.5)** 在 Mask Resolver 入口检测;违反时按 MX-3..MX-6 行为。
- 整流 / 反积 / 限幅 等具体算法由 E1 自行决定,不在 D6 范围。

### 4.7 Reset / disarmed cmd_mask 取值(强制态)

D6 锁定:在以下所有时刻 `cmd_mask` 必须 = `0x00000000`(全 0,所有位清位)。

| 时刻 | `cmd_mask` 强制值 | 来源契约 | 备注 |
|---|---|---|---|
| FMS_init 完成后第一帧 | `0` | A6 §4.4.2 + D1 §4.1.3 | HARDCODED |
| FMS reset 触发后(global reset)| `0` | A6 §4.5.1 + D2 §4.6.2 | 同帧伴随 `FMS_Out_Bus.reset = 1`(若字段存在,A6 OQ-A6-1)|
| DISARM mode(若 D1 mode 集合包含)| `0` | D1 §4.4(B2 `STATUS_DISARMED` echo)| Controller 输出零推力 + 零角速度参考 |
| FAILSAFE / `FAIL_INS_INVALID` | `0` | A6 §4.5.3 + §4.5 表 | INS 完全失效情境 |
| FAILSAFE / 其他 `FailsafeState`(链路丢失 / 低电池 / 估计器降级 / geofence)| 由 D1 §4.4 锁定 mode 子集决定;**默认走该子模式正常 cmd_mask 集** | D1 / B2 §4.4.7 | 例如 `FAIL_LINK_LOSS` → RTL 类 cmd_mask;`FAIL_INS_INVALID` → 全 0 |

**Controller 收到 `cmd_mask = 0` 的契约义务**(per MX-1):
- 输出零总推力(无论 BIT_THROTTLE_PASSTHROUGH 字段值如何 — 字段值 = 0 per A6)。
- 输出零角速度环参考。
- 各环 integrator 清零(local reset,per A6 §4.5.2)。
- 不主动启动任何环路。
- 不发 `ERR_FMS_CMD_INCONSISTENT`(全 0 是契约定义的安全态,不是错误)。

### 4.8 Cross-references(契约消费方一览)

- [B1 §4.7.1](../B-contracts/B1-bus-inventory.md) — `FMS_Out_Bus.cmd_mask` 字段 schema(uint32 bitfield;字段位置 14)
- [B1 §4.9.3](../B-contracts/B1-bus-inventory.md) — cmd_mask gated 字段对照表(D6 与 B1 双向对齐)
- [B2 §4.4.14](../B-contracts/B2-enum-inventory.md) — `cmd_mask` 位号工作名(`MASK_BIT_*`,数值 0..7,Phase 2 8 位);D6 引用,不重定义
- [B2 §4.4.10 / §4.4.11](../B-contracts/B2-enum-inventory.md) — `INS_Status` / `INS_Flag` 位号(D6 §4.5 validity gate 来源)
- [B2 §4.4.12](../B-contracts/B2-enum-inventory.md) — `ErrorCode`(D6 §4.4 / §4.3.4 / §4.3.5 触发的错误码工作名 `ERR_FMS_CMD_INCONSISTENT` / `ERR_INS_INVALID`;**具体 enum 名留 B2 / E1 锁**,D6 仅给工作名)
- [A6 §4.4.2](../A-architecture/A6-init-reset-contract.md) — `cmd_mask` reset = HARDCODED 全 0
- [A6 §4.5.2](../A-architecture/A6-init-reset-contract.md) — local reset 由 cmd_mask 部分位翻转触发,Controller 仅 reset 受影响环路
- [D3 Mode Manager](D3-mode-manager.md) (Wave 8 sibling) — Mode → cmd_mask emission table(D3 fills)
- [D4 Command Shaper](D4-command-shaper.md) (Wave 8 sibling) — Shaper → cmd_mask + setpoint 字段值对(D4 fills numerics)
- [E1 Controller 功能设计](../E-controller/E1-controller-functional.md) (Wave 8 sibling) — cmd_mask → loop trim rules(E1 fills algorithms)
- [F3 INS_Out_Bus 消费规则](../F-ins-contract/F3-consumption-rules.md) (Wave 9 forward-cite) — Controller 在 MX-3..MX-6 触发时的具体降级算法
- [B4 §4.1](../B-contracts/B4-contract-diff.md) — 契约 diff 覆盖 cmd_mask 位号数值(b01..b04 enum 检查);**位语义层差异**由 D6 review 拦截,不在 B4/I3 自动覆盖范围

## 5. 已知风险与悬而未决问题

- **Phase 3+ 位扩展不在 D6 锁定范围**
  - 影响:固定翼 / VTOL / 浮空滑行模式可能需要新位(横向 / 纵向通道分离 / 直接力 / 直接力矩);D6 当前只锁 Phase 2 8 位
  - 处置:open;Phase 3 / 4 启动时新增 D6 修订,与 B2 §4.4.14 同步扩位

- **`Auto_Cmd_Bus.cmd_mask_request` 采纳标准未锁定**
  - 影响:M-11 OFFBOARD 中 Auto 给的 mask 请求 与 FMS 仲裁后的实际 emit 之间的差异规则未量化;D1 §4.6.2 仅说"D1 决定是否采纳;D6 锁规则";D6 §4.6.1 仅说"采纳前经 §4.4 互斥过滤",未定义其他过滤规则(如安全包络 / mode 上下文)
  - 处置:open;Wave 8 D3 author 在 Stateflow 设计期与 D6 同步,可能需 D6 §4.6.1 加补充列(`mask_request_acceptance_rule`)

- **MX-2 仲裁路径在 mixed-thrust 飞行器(VTOL hover↔cruise)是否合理待定**
  - 影响:Phase 2 多旋翼 fine;Phase 3 VTOL 可能需要"部分 passthrough"语义(例如 cruise 期间 BIT_THROTTLE_PASSTHROUGH 但俯仰方向有 BIT_VEL FF)
  - 处置:open;Phase 3 时考虑分裂 BIT_THROTTLE_PASSTHROUGH 为 `BIT_THROTTLE_FORCE_PASS` / `BIT_THRUST_VECTOR_FORCE_PASS` 等更细化的位

- **MX-7 在 ground-test variant 的合法性 — 需 H1 / H4 协调**
  - 影响:MX-7(THR-only)只在 motor-test variant 出现,但 D6 当前没有 variant flag;若 H1 / H4 未来引入 ground-test 场景会触发 MX-7
  - 处置:open;[H4 故障注入](../H-verification/H4-fault-catalog.md)(Wave 11)启动时,与 D6 一并 review

- **位语义层 contract diff 在 B4 / I3 不可自动覆盖**
  - 影响:B2 锁位号数值,D6 锁位语义;若 firmware 端 cmd_mask 位 3 由"attitude" 改为"yaw rate",B4 不会发现(数值 / 名字不变,只是含义变);只能由 D6 review + 跨仓库代码 review 拦截
  - 处置:open;依赖 [B5 §4.7](../B-contracts/B5-ins-bus-mirror.md) 类似的"语义层 review" 机制(B5 锁的是 `INS_Out_Bus`;cmd_mask 是 `FMS_Out_Bus` 的字段,B4 / I3 当前 b01..b04 仅检查 enum 名 + 数值,不检查 D6 § 4.2 语义对齐);D6 后续修订时考虑加"D6 修订必须经 B4 §4.6 contract review gate" 钩子

- **`ERR_FMS_CMD_INCONSISTENT` / `ERR_INS_INVALID` 具体 enum 名待 B2 / E1 锁**
  - 影响:D6 §4.4 / §4.3.4 / §4.3.5 触发的错误码使用工作名;实际 enum 成员 B2 §4.4.12 ErrorCode 待终锁(B2 §10 changelog 标 `(B4 verify)`);Controller 端实际 emit 由 E1 决定
  - 处置:open;Wave 8 E1 author 与 B2 同步锁定;D6 §10 changelog 同步追加

## 6. 退出条件复核

对照 [`00-design-plan.md` §4.D 的 D6 行](../00-design-plan.md):退出条件 = "`cmd_mask` 位语义、参考字段优先级、互斥规则;此文件被 D 与 E 共同遵循"

| # | 退出条件原文 | 本文档依据 | 状态 |
|---|---|---|---|
| 1 | `cmd_mask` **位语义** | §4.2 逐位语义定义(8 位每一位的"置位 / 清位含义 + validity 依赖 + reset-time 取值 + 生产者 + 消费者"五段式)| 满足 |
| 2 | **参考字段优先级** | §4.3 三总则(R-Pri-1/2/3)+ §4.3.2 平移域优先级表 + §4.3.3 姿态域优先级表 + §4.3.4 ATT_QUAT vs ATT_EUL 二选一 + §4.3.5 yaw 三态裁决表 + §4.3.6 全表外→内总优先级图 | 满足 |
| 3 | **互斥规则** | §4.4 互斥规则表(MX-1..MX-10)+ §4.4.1 truth-table(合法核心组合 + 关键非法触发);Controller `Action` 列锁定行为 | 满足 |
| 4 | **此文件被 D 与 E 共同遵循** | §4.6 D3 / D4 / E1 hand-off 表(列头与契约形状);§4.7 reset / disarmed cmd_mask 强制态;§3 co-seal batch (D3, D4, D6, E1) 声明;§4.5 INS validity gate 政策(F3 forward-cite)| 满足 |

## 7. 下游影响

按 [`01-design-relationships.md` §4.4 + §4.5](../01-design-relationships.md) D 区 / E 区出边:

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [D3 Mode Manager](D3-mode-manager.md)(Wave 8 sibling)| ↔ co-seal | §4.6.1 mode → cmd_mask emission table 列头与契约形状;§4.4 互斥规则 + §4.5 INS validity 政策 → D3 Stateflow 守卫条件 |
| [D4 Command Shaper](D4-command-shaper.md)(Wave 8 sibling)| ↔ co-seal | §4.6.2 shaper → cmd_mask + setpoint 字段值配对契约;§4.3.4 ATT 二选一 + §4.4 MX-9 → D4 字段产出守卫 |
| [E1 Controller 功能设计](../E-controller/E1-controller-functional.md)(Wave 8 sibling)| ↔ co-seal | §4.6.3 cmd_mask → loop trim rules 列头与契约形状;§4.3 优先级 + §4.4 互斥 → E1 Mask Resolver 设计输入;§4.5 INS validity gate → E1 fallback hook |
| [D5 多旋翼 leaf](D5-multicopter-leaf.md)(Wave 9)| ⇢ | §4.4.1 truth-table 已锁定的合法 mode-cmd 组合 → D5 多旋翼 mode-cmd 数值表 |
| [E2 Controller 结构设计](../E-controller/E2-controller-structural.md)(已 reviewed)| ⇢(known-loose `D6 → E2`)| §4.6.3 hand-off 列头确认 E2 §4.1.1 Mask Resolver 路由槽位结构兼容(E2 显式 forward-cite Wave 8;D6 锁定后 E2 若需修订走变更日志)|
| [F3 INS_Out_Bus 消费规则](../F-ins-contract/F3-consumption-rules.md)(Wave 9)| ⇢ | §4.5 FMS 应 emit cmd_mask 政策;§4.4 MX-3..MX-6 Controller 行为 → F3 锁具体降级算法 |
| [G3 日志与可观测性](../G-harness/G3-logging.md)(Wave 10)| ⇢ | §4.2 / §4.6 cmd_mask 各位语义 → G3 logsout 必含 `cmd_mask`(per A1 §13)+ 位级 trace 列表 |
| [H2 验证指标](../H-verification/H2-metrics.md)(Wave 11)| ⇢ | §4.4 truth-table 合法 / 非法组合 → H2 验证 cmd_mask 状态正确性的判据数据 |
| [B4 契约 diff](../B-contracts/B4-contract-diff.md) / [I3](../I-tooling/I3-contract-diff.md)| (反向反馈)| §5 风险:位语义层 diff 不在 B4 / I3 自动范围;D6 修订时 INDEX 决策日志登记 |

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-08 | Wave 8 D6 author | 初稿 — 锁定 8 位 cmd_mask 语义(§4.2)+ 三层优先级(§4.3.2/4.3.3/4.3.5)+ 10 条互斥规则(§4.4)+ §4.4.1 truth-table 14 行(合法核心 + 关键非法)+ §4.5 INS validity 政策 + §4.6 D3/D4/E1 hand-off 列头与契约形状 + §4.7 reset/disarmed 强制 cmd_mask=0 + co-seal batch (D3, D4, D6, E1) 声明 |

## Self-check

- [x] frontmatter 完整,字段值合法
- [x] 上游文档全部存在且 status ≥ reviewed,**或**为本工作项所在 co-seal batch 的同批兄弟(允许同批互相引用,需在 §3 注明 batch 名)— D3 / D4 / E1 标记 `draft, co-seal batch (D3, D4, D6, E1)`,符合 [RULES §6 self-check item 2](../RULES.md);其余上游(架构 v1 / A3 / A6 / B1 / B2 / D1 / D2 / E2)全部 reviewed
- [x] 退出条件逐条复核完成,每条均给出依据(§6 表 4 行,与 [00-design-plan.md §4.D D6](../00-design-plan.md) 三项退出条件 + "此文件被 D 与 E 共同遵循" 声明逐条对应)
- [x] 引用路径全部可点击访问(架构 v1 / A2 / A3 / A6 / B1 / B2 / B4 / D1 / D2 / D3 / D4 / D5 / E1 / E2 / F3 / G3 / H2 / I3 均使用相对路径)
- [x] 不存在 §5 禁则中的内容(无 `.slx` 截图、无可执行 `.m` 代码、无 firmware 实现细节复述、无重复定义 bus/enum 字段;cmd_mask 位号 / 名 / 数值由 B2 拥有,D6 仅引用)
- [ ] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记 — **未勾选;留待 orchestrator 在 INDEX 登记时勾选**(per Wave 8 batch 协议)
- [ ] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径 — `<pending hash>` 占位符(per established batch convention,with A3/A6/A7/B1/B2/B3/B5 同期补齐)
- [x] 下游影响已沿关系图识别完毕(§7 表覆盖 [01 §4.4 D 区出边](../01-design-relationships.md) D6 ↔ D3/D4 + [§4.5 E 区出边](../01-design-relationships.md) D6 → E1 + 衍生 D5 / E2 known-loose / F3 / G3 / H2 / B4 反馈)
- [x] 文档不超出本工作项范围(无越权设计:不重定义 cmd_mask 位号 / 名 / 数值 — 留 B2;不设计 Stateflow 状态 / 转换 / 守卫 — 留 D3;不写 shaper 整形公式 — 留 D4;不锁 Controller 各环算法 / cascade 实现 — 留 E1;不锁 Controller fallback 算法 — 留 F3;不锁 mixer / 分配矩阵 — 留 E4;只锁 cmd_mask 位语义 + 优先级 + 互斥 + hand-off 契约形状)
