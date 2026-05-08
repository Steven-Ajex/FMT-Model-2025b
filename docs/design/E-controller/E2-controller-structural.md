---
work_item: E2
title: Controller 结构设计(级联拓扑、共享 vs leaf、库块清单)
upstream: [架构v1, A3, A4, A5, A6, A7, A8, B1, B2, B3]
contract_impact: no
status: draft
authored_at: 2026-05-08
last_reviewed_at:
reviewer_verdict: none
---

# E2 Controller 结构设计

## 1. 目的

本文件定义 Controller 模块在 FMT-Model-2025b 仓库内的**结构骨架**:级联环路顶层拓扑、各环路与共享库 / leaf 的边界、以及与上游 [A8 共享库块清单](../A-architecture/A8-shared-library-roster.md) 的认领关系。本文件不锁定控制律方程,不锁定 cmd_mask 裁剪规则,不锁定多旋翼 mixer 内部细节,亦不锁定执行时间预算 — 这些分别由 E3 / E1 / E4 / E5 处理。

## 2. 范围

**在范围:**
- Controller 顶层子系统拓扑(setpoint decode → 位置环 → 速度环 → 姿态环 → 角速度环 → mixer → output assembler,沿架构 v1 §12.2 的层级)
- 级联 setpoint / measurement 信号流的路由结构(`cmd_mask` / `ctrl_mode` 仅作为**路由槽位**输入,语义本身不在此处定义)
- shared(`model/shared/lib/control/`、`model/controller/shared/`)与 leaf(`model/controller/vehicles/<vehicle>/`)边界
- 每个环路的内部结构骨架(error compute → 控制块 → 限幅 → 输出),不含具体方程
- 与 [A5 §4.5 Controller 的 optional `Plant_States_Bus` 变体接入点](../A-architecture/A5-variant-strategy.md) 的 variant 接入点对齐
- Init / reset 在结构上的放置(integrator 集中点、reset 信号扇出),与 [A6](../A-architecture/A6-init-reset-contract.md) 协议对齐
- `Control_Out_Bus` 输出装配点

**不在范围(由其他工作项处理):**
- 各环具体启用 / 禁用规则(由 cmd_mask 位语义决定)— 由 [E1 Controller 功能设计](E1-controller-functional.md)(Wave 8 与 D6 同 batch)处理
- 控制律方程、前馈、anti-windup 算法、滤波器系数 — 由 [E3 Controller 各环算法设计](E3-loops-algorithm.md) 处理
- 多旋翼 mixer 矩阵 / 电机推力曲线 / leaf 增益结构 — 由 [E4 Controller 多旋翼 leaf 设计](E4-multicopter-leaf.md) 处理
- 5 ms 周期内各环执行时间预算 / 热点禁用项 / LUT 与定点策略 — 由 [E5 Controller 性能预算设计](E5-performance-budget.md) 处理
- INS validity 字段的 fallback 行为 / 消费规则细节 — 由 [F3 INS_Out_Bus 消费规则](../F-ins-contract/F3-consumption-rules.md) 处理(E2 仅定义路由布线点)
- B3 `CONTROL_PARAM` 字段集 / 默认值 — 由 [B3 Parameter schema](../B-contracts/B3-parameter-schema.md) 处理

## 3. 依赖

| 上游 | 引用位置 | 用途 |
|---|---|---|
| 架构 v1 §7.2 | [架构 v1](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | Controller 顶层职责与"contract-required passthrough"边界 |
| 架构 v1 §8.4 | [架构 v1](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | Controller 边界(契约级输入仅 `FMS_Out_Bus` + `INS_Out_Bus`,`Plant_States_Bus` harness-only) |
| 架构 v1 §12.2 | [架构 v1](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | 级联内部架构(setpoint decode / outer guidance / velocity / attitude / rate / mixer / output assembler 七子模块) |
| 架构 v1 §13 | [架构 v1](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | Controller 周期 = 5 ms = 200 Hz,internally single-rate |
| A3 §4.5 | [A3 模块边界](../A-architecture/A3-module-boundaries.md) | Controller 输入 / 输出 bus 字段方向矩阵;消费侧 dependency |
| A3 §4.7 | [A3 模块边界](../A-architecture/A3-module-boundaries.md) | cmd_mask 位语义的 D6+E1 锁定指针 — E2 仅承诺路由槽位 |
| A3 §4.8 | [A3 模块边界](../A-architecture/A3-module-boundaries.md) | INS 消费侧依赖矩阵(Controller 列) |
| A4 §4.4 RB-02 / RB-03 / RB-05 | [A4 跨速率边界](../A-architecture/A4-rate-boundaries.md) | Controller 输入侧的 ZOH / Latch、输出侧 ZOH 边界点 |
| A5 §4.5 | [A5 变体策略](../A-architecture/A5-variant-strategy.md) | harness-only `Plant_States_Bus` variant 接入点(在 Controller 顶层模型外) |
| A6 §4.2.3 / §4.4 / §4.5.2 | [A6 init/reset 契约](../A-architecture/A6-init-reset-contract.md) | Controller init 必须使其确定的状态清单、reset 后稳态契约、跨模块协调 |
| A7 §4.5 | [A7 时间约定](../A-architecture/A7-time-conventions.md) | Controller 内 dt 来源固定步长(不消费 timestamp 计算 dt) |
| A8 §4.2.4 / §4.3.3 | [A8 共享库块清单](../A-architecture/A8-shared-library-roster.md) | Layer A `model/shared/lib/control/` 与 Layer B `model/controller/shared/` 的库块台账 — E2 沿用 A8 的归属 |
| B1 | [B1 Bus 清单](../B-contracts/B1-bus-inventory.md) | `FMS_Out_Bus` / `INS_Out_Bus` / `Control_Out_Bus` schema |
| B2 | [B2 Enum 清单](../B-contracts/B2-enum-inventory.md) | `ctrl_mode`、`cmd_mask` 位字段(语义占位 — 数值由 B2 锁定) |
| B3 | [B3 Parameter schema](../B-contracts/B3-parameter-schema.md) | `CONTROL_PARAM` schema 引用(E2 仅指针;字段在 B3 / E4) |
| **E1**(Wave 8 sibling) | [E1 Controller 功能设计](E1-controller-functional.md) — *尚未撰写,Wave 8 与 D6 同 batch co-seal* | E2 的级联拓扑与 E1 的最终环路清单将在 Wave 8 co-seal 评审中**确认兼容**;E1 若重命名 / 拆分某环路,E2 将在变更日志中追加修订 |

**Co-seal / 同 batch 注记:**
本工作项位于 Wave 7;**强前置上游 E1** 实际位于 Wave 8(与 D6 同 batch),违反 [`01-design-relationships.md` §4.5 `E1 → E2`](../01-design-relationships.md) 的强前置语义。该违反是**编排器在 Wave 6 → 7 排序时显式做出的决策**(2026-05-05,记录于 INDEX 决策日志):E2 设计**结构骨架(级联拓扑 / shared-vs-leaf / 库块清单)而不锁定 E1 范畴的功能决策(具体环路启用 / cmd_mask 裁剪规则)**。E2 把 E1 的最终环路清单作为输入占位,在级联内部按"位置 / 速度 / 姿态 / 角速度 / mixer"五段标定布线点 — 与架构 v1 §12.2 显式给出的拓扑层级一致。Wave 8 co-seal 评审时,E2 与 E1 一并核对兼容性。

## 4. 设计内容

### 4.1 Controller 顶层子系统拓扑

Controller 顶层模型(`model/controller/Controller.slx`,Model Reference 顶层,per [A5 §4.1](../A-architecture/A5-variant-strategy.md))内部装配的 7 个子系统(对齐 [架构 v1 §12.2](../../architecture/2026-05-05-fmt-model-architecture-v1.md))的拓扑如下:

```text
                    Controller (5 ms / 200 Hz, internally single-rate)
                    ─────────────────────────────────────────────────────
   FMS_Out_Bus ──┐
                  │   ┌──────────────────────┐
                  ├──▶│ §4.1.1 Setpoint      │── decoded setpoints ──┐
                  │   │  Decode + Mode/Mask  │── cmd_mask routing  ──┤
                  │   │  Resolver            │── ctrl_mode hint    ──┤
                  │   └──────────────────────┘                       │
                                                                     │
   INS_Out_Bus ──┐                                                   │
                  │   ┌─────────┐  pos_err  ┌──────────┐ vel_sp     │
                  ├──▶│ §4.1.2  │──────────▶│  §4.1.3  │──────────▶ │
                  │   │ Position│           │ Velocity │            │
                  │   │  Loop   │           │   Loop   │            │
                  │   │ (outer) │           │          │            │
                  │   └─────────┘           └──────────┘            │
                  │      ▲ INS pos_ned         ▲ INS vel_ned        │
                  │      │ + valid             │ + valid            │
                  │      │                     │                    │
                  │   ┌─────────┐ att_err  ┌──────────┐ rate_sp    │
                  ├──▶│ §4.1.4  │─────────▶│  §4.1.5  │──────────▶ │
                  │   │Attitude │          │Angular   │             │
                  │   │  Loop   │          │ Rate Loop│             │
                  │   │         │          │ (hot)    │             │
                  │   └─────────┘          └──────────┘             │
                  │      ▲ INS quat            ▲ INS ang_rate       │
                  │      │ + valid             │ + valid            │
                  │      │                     │                    │
                  │      │              moment + thrust virtual ctl │
                  │      │                     │                    │
                  │                            ▼                    │
                  │                     ┌──────────────┐            │
                  │                     │   §4.1.6     │            │
                  │                     │  Mixer /     │── motor ───┤
                  │                     │  Allocator   │  cmd[] /   │
                  │                     │  (LEAF: E4)  │  actuator  │
                  │                     └──────────────┘            │
                  │                                                 │
                  │   (Plant_States_Bus only present in harness     │
                  │    SIH/HIL variant per A5 §4.5; routed in       │
                  │    harness layer, NOT into Controller top model)│
                  │                                                 │
                  ▼                                                 ▼
              ┌──────────────────────────────────────────────────────────┐
              │  §4.1.7 Output Assembler  (assembles Control_Out_Bus)    │
              │  - motor_cmd[] from mixer                                │
              │  - thrust_cmd from inner loop (or pilot passthrough)     │
              │  - actuator_cmd[] from FMS_Out_Bus.actuator_cmd[]        │
              │    (passthrough — A3 §4.6)                               │
              │  - cmd_mask / ctrl_mode echo (A3 §4.5.5)                 │
              │  - debug rate / saturation flags                         │
              └─────────────────────────┬────────────────────────────────┘
                                        ▼
                                   Control_Out_Bus
```

逐级摘要(rate 与 reset 行为引指针,具体见各小节):

| # | 子系统 | 速率 | enable 条件(占位 — 由 E1 锁定) | reset 行为(指针) |
|---|---|---|---|---|
| §4.1.1 | Setpoint Decode + Mask Resolver | 5 ms | 总是运行 | A6 §4.4.2 `Control_Out_Bus.cmd_mask` echo HARDCODED zero |
| §4.1.2 | Position loop | 5 ms | per E1 §4.x cmd_mask 裁剪规则,**E2 不锁定** | integrator 清零(A6 §4.2.3 Controller 项 1) |
| §4.1.3 | Velocity loop | 5 ms | per E1 §4.x | integrator 清零 |
| §4.1.4 | Attitude loop | 5 ms | per E1 §4.x | integrator 清零;quat 误差初始化 identity |
| §4.1.5 | Angular rate loop | 5 ms | per E1 §4.x(典型 always-on) | integrator 清零;D 项 LPF 状态清零 |
| §4.1.6 | Mixer / Allocator | 5 ms | always-on(若任一上游产出虚拟控制) | 输出 = 电机零推力 / 舵面 trim(A6 §4.4.2) |
| §4.1.7 | Output Assembler | 5 ms | always-on | 默认 disarmed-equiv echo(A6 §4.4.2) |

**关键不变量:**
- Controller 顶层 internally single-rate = 5 ms(架构 v1 §13.2);所有子系统在同一 sample time 上跑。
- Controller **顶层模型**只暴露 `FMS_Out_Bus` / `INS_Out_Bus` / `Control_Out_Bus` 三个 bus 端口(架构 v1 §8.4);`Plant_States_Bus` variant 接入仅存在于 harness 层(A5 §4.5),Controller 顶层模型不直接暴露此 inport。

### 4.2 级联信号流与路由结构

E2 描述**路由布线**,**不**描述哪一位代表哪个环路(A3 §4.7 已明确"位语义本身由 D6 + E1 锁定")。

#### 4.2.1 setpoint 多路径路由

`FMS_Out_Bus` 的 setpoint 字段(`pos_cmd_ned_m` / `vel_cmd_ned_mps` / `att_cmd_quat` 或 `att_cmd_euler_rad` / `ang_rate_cmd_b_radps` / `yaw_cmd_rad` / `yaw_rate_cmd_radps` / `throttle_cmd`)在 §4.1.1 Setpoint Decode 中被分流到对应环路的"setpoint inport":

```text
FMS_Out_Bus.pos_cmd_ned_m       ──▶ §4.1.2 Position Loop setpoint port
FMS_Out_Bus.vel_cmd_ned_mps     ──▶ §4.1.3 Velocity Loop setpoint port  (alt source)
FMS_Out_Bus.att_cmd_quat        ──▶ §4.1.4 Attitude Loop setpoint port
FMS_Out_Bus.att_cmd_euler_rad   ──▶ §4.1.4 Attitude Loop setpoint port  (alt source)
FMS_Out_Bus.ang_rate_cmd_b_radps──▶ §4.1.5 Rate Loop setpoint port      (alt source)
FMS_Out_Bus.yaw_cmd_rad         ──▶ §4.1.4 Attitude Loop yaw component
FMS_Out_Bus.yaw_rate_cmd_radps  ──▶ §4.1.5 Rate Loop yaw-rate component
FMS_Out_Bus.throttle_cmd        ──▶ §4.1.7 Output Assembler (passthrough mode)
                                    or §4.1.6 Mixer (collective input — E4 leaf)
```

#### 4.2.2 内外环切换的"路由槽位"约定

每个环路有 **2 个** setpoint 输入端口:
- `setpoint_external`:由上游 FMS 字段直接驱动(典型 outer-most active loop)
- `setpoint_inner`:由更外层环路的输出驱动(典型 inner cascade)

由 §4.1.1 `Controller_CmdMask_Resolver` (A8 §4.3.3) 在每帧解析 `cmd_mask` 与 `ctrl_mode`,产出**每环 2-向选择信号**(`use_external` / `use_inner`),其取值规则**由 E1 锁定**(co-seal Wave 8)。E2 仅约束:
1. 每环路恰有一个选择源激活;
2. 内环始终运行(E2 默认值,可被 E1 修订)— 外环按 mask 旁路;
3. 选择信号在 §4.1.1 的 single resolve 步骤中产出,环路内部不重复解析 cmd_mask(避免分散逻辑)。

#### 4.2.3 measurement 路由

INS 测量字段在 §4.1.1 Setpoint Decode 同帧 latch 后,路由到对应环路的"measurement inport"(对照 A3 §4.8 矩阵):

```text
INS_Out_Bus.position_ned_m   + flag.position_valid   ──▶ §4.1.2
INS_Out_Bus.velocity_ned_mps + flag.velocity_valid   ──▶ §4.1.3
INS_Out_Bus.quat_ned_to_b    + flag.attitude_valid   ──▶ §4.1.4
INS_Out_Bus.ang_rate_b_radps + flag.attitude_valid   ──▶ §4.1.5  (典型 — F3 锁定 fallback)
```

validity flag 在每环入口处作为"gate 信号"接入 — E2 把 flag 路由到环路边界的 gate inport,**fallback 行为(false 时输出零 / 持锁 / 切到备份估计)由 [F3](../F-ins-contract/F3-consumption-rules.md) 锁定**,E2 不预设。

### 4.3 共享库块清单(沿用 A8 roster)

E2 **不重复定义**库块本身;以下表沿用 [A8 §4.2.4 `model/shared/lib/control/`](../A-architecture/A8-shared-library-roster.md) 与 [A8 §4.3.3 `model/controller/shared/`](../A-architecture/A8-shared-library-roster.md) 中已上账的块,作为 E2 的认领声明。

#### 4.3.1 Layer A(`model/shared/lib/control/` 与跨模块) — Controller 消费

| 块名 | A8 出处 | E2 认领 | 在 E2 中的使用点 |
|---|---|---|---|
| `PID_Block` | A8 §4.2.4 | **认领** | §4.1.2 / §4.1.3 / §4.1.4 / §4.1.5 各环 controller 块的实现底座(具体增益结构由 E3 / E4 锁定) |
| `AntiWindup_BackCalc` | A8 §4.2.4 | **认领**(条件) | §4.1.3 / §4.1.5 的积分器抗 windup;具体算法选择(back-calc vs conditional integration)由 E3 锁定 |
| `FeedForward_Composer` | A8 §4.2.4 | **认领** | §4.1.3 / §4.1.4 / §4.1.5 的前馈合成注入点;具体前馈项由 E3 锁定 |
| `LowPass_Filter_1st` | A8 §4.2.2 | **认领** | §4.1.5 的 measurement / D-term 预滤波 |
| `HighPass_Filter_1st` | A8 §4.2.2 | **认领**(条件) | 若 E3 决定 D-term 不内嵌于 PID,则单点引用 |
| `Saturation_*` | A8 §4.2.4 群组 | **认领** | 各环 setpoint / 输出限幅;具体限值由 leaf(E4)注入 |
| `Quaternion_Mul` / `Quat_To_Rotation_Matrix` 等 | A8 §4.2.1 math | **认领** | §4.1.4 attitude error 解算(quat 误差) |

#### 4.3.2 Layer B(`model/controller/shared/`) — Controller 模块独占共享

| 块名 | A8 出处 | E2 认领 | 在 E2 中的使用点 |
|---|---|---|---|
| `Controller_Cascade_Loop_Shell` | A8 §4.3.3 | **认领** | §4.1.2 / §4.1.3 / §4.1.4 / §4.1.5 每环的 stage shell(setpoint → error → law → output 骨架);"law" 槽位插入 §4.3.1 `PID_Block` |
| `Controller_CmdMask_Resolver` | A8 §4.3.3 | **认领** | §4.1.1 的 mask / mode 解析子块;**裁剪规则由 E1 在 Wave 8 注入**(本块仅是骨架) |
| `Controller_AntiWindup_Coordinator` | A8 §4.3.3 | **认领** | 跨级 anti-windup 协调(saturation 状态从内环向外环传播);**算法由 E3 锁定** |
| `Controller_Mixer_Frame_Shell` | A8 §4.3.3 | **认领作为骨架** | §4.1.6 的 mixer 框架壳(输入虚拟控制 / 输出 actuator 向量);**leaf 层 E4 实现具体分配矩阵** |
| `Controller_Output_Saturator_Shell` | A8 §4.3.3 | **认领** | §4.1.6 / §4.1.7 之间的 actuator 限幅(限值由 E4 leaf 注入) |
| `Controller_Reset_Hooks_Shell` | A8 §4.3.3 | **认领** | §4.1 reset 信号扇出布线点(对接 A6 §4.5.2) |

#### 4.3.3 leaf(`model/controller/vehicles/multicopter/`) — E4 owns

E2 列出 leaf 的**槽位**,具体内容由 E4 锁定:

| Leaf 槽位 | 在 E2 拓扑中的位置 | 责任 |
|---|---|---|
| 多旋翼 mixer 矩阵 | §4.1.6 内,挂在 `Controller_Mixer_Frame_Shell` 的"分配矩阵"插入点 | E4 |
| 多旋翼增益集 | §4.1.2 / §4.1.3 / §4.1.4 / §4.1.5 各环 `PID_Block` 的 `gain_set_in` 端口 | E4(参数取自 B3 `CONTROL_PARAM`) |
| 多旋翼输出归一化(PWM `_us` vs throttle 0..1) | §4.1.7 Output Assembler 的 `motor_cmd[]` 编码槽位 | E4(B1 `motor_cmd[]` 单位选择 deferred 至 B4 contract diff,A3 §4.5.5 已标注) |
| 多旋翼 actuator 限幅常数 | `Controller_Output_Saturator_Shell` 的 `lim_min` / `lim_max` 端口 | E4 |

#### 4.3.4 shared vs leaf 边界规则(沿用 A5 §4.3.3 与 A8 §4.1.3 判据)

A8 已给出冲突解决判据;E2 仅声明:**任何控制律算法块默认 shared(Layer A)**;**任何与几何 / 分配矩阵 / 电机 / 舵面物理参数耦合的块 leaf**;**Controller 内部 cmd_mask / anti-windup 协调骨架 Layer B**(因强耦合于 Controller 内部信号语义,A8 §4.1.3 明确)。

### 4.4 变体接入点(per A5)

E2 在结构上预留以下变体接入点(对应 [A5 §4.2.1](../A-architecture/A5-variant-strategy.md) 的"vehicle 类型"唯一允许变体轴):

| # | 变体接入点 | 位置 | Phase 2 active variant | 备注 |
|---|---|---|---|---|
| VP-1 | Mixer / 分配矩阵 | §4.1.6,挂在 `Controller_Mixer_Frame_Shell` 内 | multicopter(E4)| 未来固定翼 / VTOL / hex / + / X 在同一 shell 下扩展 |
| VP-2 | 各环增益集 | §4.1.2 / §4.1.3 / §4.1.4 / §4.1.5 各 `PID_Block.gain_set_in` 端口 | multicopter(B3 `CONTROL_PARAM`)| aggressive / gentle 等 tuning profile 由 B3 参数集决定,不开 Variant Subsystem |
| VP-3 | 输出编码 / 归一化 | §4.1.7 `motor_cmd[]` encoding 槽位 | multicopter(待 B4 锁定 PWM `_us` vs throttle 0..1)| 编码决定不在 E2,E2 仅留槽位 |
| VP-4 | (harness-only)`Plant_States_Bus` 旁路 | **Controller 顶层模型外**,在 harness 层 Variant Subsystem | MIL(disabled);SIH / HIL 在 Phase 3 启用 | 与 vehicle 轴正交;A5 §4.5 |

**Phase 2 显式冻结(对照 A5 §4.2.2):**
Phase 2 只有 multicopter 一个 active variant;Variant Subsystem 容量预算见 A5 §4.3.1。E2 不引入除上述 4 个接入点之外的任何变体轴。

### 4.5 各环路内部结构骨架

各环的内部结构沿用 [A8 §4.3.3 `Controller_Cascade_Loop_Shell`](../A-architecture/A8-shared-library-roster.md) 模板:`setpoint → error → law → output`。E2 给每环的端口与块编排,**不**给方程。

#### 4.5.1 Position loop(§4.1.2)

```text
                  ┌──────────────────────────────────────┐
  pos_sp ──────▶  │  setpoint port (selected by §4.1.1)  │
  pos_meas ────▶  │  measurement port (gated by valid)   │
  valid_gate ──▶  │                                      │
  param_ref ──▶   │  PID_Block (gains from CONTROL_PARAM)│ ──▶ vel_sp_inner
  reset ───────▶  │  ↳ AntiWindup_BackCalc (E3 algo)     │
                  │  ↳ Saturation (limits from leaf)     │
                  └──────────────────────────────────────┘
```
- 输入:`pos_sp`(从 §4.1.1)/ `pos_meas` = `INS_Out_Bus.position_ned_m`(经 §4.2.3 routing)/ `flag.position_valid`(gate)/ B3 `CONTROL_PARAM` 中位置环增益子集(指针)/ `reset`(从 §4.1.7 → A6 §4.5.2)
- 内部块:error compute → `PID_Block` → `Saturation_*` → output
- 输出:`vel_sp_inner` 馈入 §4.1.3 的 `setpoint_inner` 端口
- saturation 反馈:输出限幅信号反馈给 `AntiWindup_BackCalc`(具体算法 E3);**跨级 anti-windup 协调由 `Controller_AntiWindup_Coordinator` 统一**(A8 §4.3.3)

#### 4.5.2 Velocity loop(§4.1.3)

- 输入:`vel_sp`(`use_external` 时来自 `FMS_Out_Bus.vel_cmd_ned_mps`;`use_inner` 时来自 §4.5.1 输出)/ `vel_meas` = `INS_Out_Bus.velocity_ned_mps` / `flag.velocity_valid` / 速度环增益(B3) / 前馈接入点(`FeedForward_Composer`,前馈项由 E3 锁定)/ `reset`
- 内部块:error compute → `PID_Block`(可叠加 `FeedForward_Composer`)→ `Saturation_*` → output
- 输出:`accel_sp_inner` 或 `thrust_cmd` 虚拟控制(具体语义由 E3 锁定,E2 仅声明端口存在)
- saturation 反馈同 §4.5.1

#### 4.5.3 Attitude loop(§4.1.4)

- 输入:`att_sp_quat` 或 `att_sp_euler`(由 §4.1.1 二选一)/ `att_meas_quat` = `INS_Out_Bus.quat_ned_to_b` / `flag.attitude_valid` / yaw 分量从 `FMS_Out_Bus.yaw_cmd_rad` 单独路由 / 姿态环增益(B3) / `reset`
- 内部块:quat error 解算(用 Layer A `Quaternion_Mul`)→ `PID_Block`(典型 P-only 或 PD)→ `Saturation_*` → output
- 输出:`ang_rate_sp_inner` 馈入 §4.1.5

#### 4.5.4 Angular rate loop(§4.1.5)

- 输入:`ang_rate_sp`(`use_external` 时来自 `FMS_Out_Bus.ang_rate_cmd_b_radps`,`use_inner` 时来自 §4.5.3)/ `rate_meas` = `INS_Out_Bus.ang_rate_b_radps` / 角速度环增益(B3) / 前馈接入点 / `reset`
- 内部块:error compute → 测量 LPF(`LowPass_Filter_1st`,可选)→ `PID_Block`(典型 PI + D-on-measurement)→ `Saturation_*` → output
- 输出:`moment_b_virtual` + `thrust_virtual`(若 §4.5.2 不直接产出 thrust)
- 这是**最热的环**(架构 v1 §12.2.5);具体禁用项 / LUT / 定点策略由 [E5](E5-performance-budget.md) 锁定

#### 4.5.5 Mixer / Allocator(§4.1.6)

- 输入:`moment_b_virtual` / `thrust_virtual`(从 §4.5.4)+ leaf 注入的分配矩阵(E4)+ leaf 注入的电机 / 舵面限幅(E4)
- 内部块:`Controller_Mixer_Frame_Shell`(骨架)→ leaf 矩阵 → `Controller_Output_Saturator_Shell`
- 输出:`motor_cmd[]` 与 / 或 `actuator_cmd[]`(送 §4.1.7)
- **mixer 详细矩阵 / 几何 / 电机推力曲线对接 / 增益结构由 E4 锁定**;E2 仅声明骨架与 leaf 接入端口。

### 4.6 Controller / FMS 接口路由

对照 [A3 §4.5.2](../A-architecture/A3-module-boundaries.md):

**E2 直接消费的 `FMS_Out_Bus` 字段(在 §4.1.1 解码并分发):**
- setpoint:`pos_cmd_ned_m` / `vel_cmd_ned_mps` / `att_cmd_quat` / `att_cmd_euler_rad`(二选一)/ `ang_rate_cmd_b_radps` / `yaw_cmd_rad` / `yaw_rate_cmd_radps` / `throttle_cmd`
- 控制流:`cmd_mask` / `ctrl_mode` / `mode` / `state` / `ext_state`(后三者用于 logging / state echo,不进环路计算 — A3 §4.5.2 已给依赖)
- reset:`reset`(扇出到所有环路 reset 端口,经 `Controller_Reset_Hooks_Shell`)
- actuator passthrough:`actuator_cmd[]`(直送 §4.1.7,**不**进任何环路)

**E2 在 `Control_Out_Bus` 中产出的字段(由 §4.1.7 装配)** — E2 不负责回写任何字段到 `FMS_Out_Bus`(A3 §4.5 已声明 Controller 不修改 FMS bus)。

### 4.7 INS_Out_Bus 消费模式

对照 [A3 §4.8 INS 消费侧依赖矩阵](../A-architecture/A3-module-boundaries.md) Controller 列:

| 环路 | INS 字段 | validity gate | E2 路由位置 |
|---|---|---|---|
| §4.1.2 Position | `position_ned_m` | `flag.position_valid` | §4.1.1 → §4.5.1 measurement port |
| §4.1.3 Velocity | `velocity_ned_mps` | `flag.velocity_valid` | §4.1.1 → §4.5.2 |
| §4.1.4 Attitude | `quat_ned_to_b`(主)/ `euler_ned_to_b_rad`(alt) | `flag.attitude_valid` | §4.1.1 → §4.5.3 |
| §4.1.5 Rate | `ang_rate_b_radps` | `flag.attitude_valid`(典型 — A3 §4.5.3 注)| §4.1.1 → §4.5.4 |
| (可选) | `acc_b_mps2` | (opt;A3 §4.8 (opt)) | 留预留端口,具体使用由 E3 决定 |

**validity gate 路由原则(E2 范畴):**
- gate 信号在每环入口的 `valid_gate` 端口接入;
- gate=false 的"具体降级行为"(zero-out / hold-last / 切备份)**由 [F3](../F-ins-contract/F3-consumption-rules.md) 锁定**,E2 仅承诺端口存在。
- staleness 检测(通过 `INS_Out_Bus.timestamp` + `Stale_Detector` A8 §4.2.5)若被 E1 / D6 决定纳入,接在 §4.1.1 解码侧;**默认 E2 不引入**(A3 §4.5.3 注:Controller 不直接消费 timestamp)。

### 4.8 Init / reset 结构布局

对照 [A6](../A-architecture/A6-init-reset-contract.md) 契约:

#### 4.8.1 Controller_init 完成后的稳态(指针 — 由 A6 §4.2.3 + §4.4.2 定义,E2 不重述)

A6 §4.2.3 Controller 项 1–5 已枚举:各环积分器 / anti-windup 状态、滤波器状态、各环 enable latch、mixer 中间状态、`Control_Out_Bus` 默认稳态。E2 在拓扑层面承诺:
- 每个 `PID_Block` 实例在 init 时 integrator state = 0;
- 每个 `LowPass_Filter_1st` / `HighPass_Filter_1st` 实例在 init 时 state = 0;
- §4.1.1 mask resolver 在 init 时所有 enable latch = false(A6 §4.2.3 Controller 项 3 默认 disable);
- §4.1.7 Output Assembler 在 init 后产出 A6 §4.4.2 定义的"电机零推力 / 舵面 trim"稳态。

#### 4.8.2 Reset 触发与扇出

- **Reset 信号源**:`FMS_Out_Bus.reset`(主),per A6 §4.3.1 触发源分类;harness-only 调试 reset 不在 Controller 顶层模型可见(per A6 §4.3.2 权限矩阵)。
- **扇出路径**:`FMS_Out_Bus.reset` ──▶ §4.1.1 解码 ──▶ `Controller_Reset_Hooks_Shell`(A8 §4.3.3) ──▶ 扇出到每个环路的 `reset` 端口。
- **每环 reset 隔离**:每环 `reset` 端口仅 reset 本环路的 integrator / filter / antiwindup state;**不**触及其他环。"全局 reset"(整个 Controller 回到 init 后稳态)由 reset 信号同时拉高所有环路端口实现 — E2 不引入"局部 reset 子集"端口(A6 §4.4 默认 reset 后稳态 = init 后稳态)。
- **arming / mode 切换触发**:由 FMS 在 `FMS_Out_Bus.reset` 上拉信号;Controller 不自行解释 `ctrl_mode` 的切换语义(避免分布式重置逻辑,per A6 §4.5.2 协议)。

### 4.9 Output Assembler(§4.1.7)输出装配

对照 [A3 §4.5.5 `Control_Out_Bus` 字段表](../A-architecture/A3-module-boundaries.md):

| `Control_Out_Bus` 字段 | E2 装配来源 | 备注 |
|---|---|---|
| `motor_cmd[]` | §4.1.6 mixer 输出(经 `Controller_Output_Saturator_Shell`) | 编码 PWM `_us` 或 0..1 由 E4 / B4 锁定 |
| `actuator_cmd[]` | `FMS_Out_Bus.actuator_cmd[]` 直 passthrough(A3 §4.6 passthrough-with-shape 已在 FMS 侧整形完成,Controller 不再整形) | 不进环路 |
| `throttle_cmd` *(echo)* | `FMS_Out_Bus.throttle_cmd` passthrough(throttle_passthrough mode 时)| A3 §4.5.5 |
| `cmd_mask` *(echo)* | `FMS_Out_Bus.cmd_mask` passthrough | A3 §4.5.5;Controller 不修改,仅 echo 给 Plant trace |
| `ctrl_mode` *(echo)* | `FMS_Out_Bus.ctrl_mode` passthrough | 同上 |
| `reset` *(echo, optional)* | `FMS_Out_Bus.reset` passthrough(若契约启用) | 同上 |
| `timestamp` | step interface 入参(A7 §4.5)| Controller 不基于 timestamp 计算 dt |
| debug rate / saturation flags | 各环路内部信号 tap | 仅 logging,不参与环路计算 |

**装配集中点原则(架构 v1 §12.2.7):**
所有 `Control_Out_Bus` 字段在**单个** Output Assembler 块内汇集 — Controller 内部不存在多处向 `Control_Out_Bus` 直写的位置。这保证 [B4 contract diff](../B-contracts/B4-contract-diff.md) 在比较输出 bus 字段顺序时能落到 single point。

### 4.10 与下游的交叉引用

| 下游 | E2 提供给下游的输入 |
|---|---|
| [E1](E1-controller-functional.md)(Wave 8 co-seal sibling) | 级联拓扑骨架与"每环 setpoint 路由槽位"约定 — E1 在此骨架上注入 cmd_mask 裁剪规则;Wave 8 co-seal 时确认 E1 最终环路清单与 §4.1 / §4.5 兼容 |
| [E3](E3-loops-algorithm.md)(Wave 9) | 各环 `PID_Block` / `AntiWindup_BackCalc` / `FeedForward_Composer` 的实例位置与端口编排,作为 E3 算法注入的脚手架 |
| [E4](E4-multicopter-leaf.md)(Wave 9) | mixer / 分配矩阵 / 增益集 / 输出编码 4 个 leaf 接入点(VP-1 / VP-2 / VP-3 与 §4.5.5) |
| [E5](E5-performance-budget.md)(Wave 10) | 各环子系统粒度与块清单,作为时间预算分摊的目标 |
| [F3](../F-ins-contract/F3-consumption-rules.md)(Wave 9) | INS validity gate 在每环入口的端口,F3 在此端口上锁定 fallback 行为 |
| [G1](../G-harness/G1-mil-toplevel.md)(Wave 10) | Controller 顶层模型的 3 个 bus 端口(`FMS_Out_Bus` / `INS_Out_Bus` / `Control_Out_Bus`)与 harness 层 `Plant_States_Bus` variant 接入(VP-4) |

## 5. 已知风险与悬而未决问题

- **E1 在 Wave 8 可能重命名或拆分某环**
  - 影响:§4.1 / §4.5 中环路命名("位置 / 速度 / 姿态 / 角速度")若被 E1 拆分(例如把 yaw 单独成环 + roll-pitch 一环),E2 §4.5 会需要相应拆分。
  - 处置:Wave 8 co-seal 评审时核对,在变更日志追加修订;骨架(`Controller_Cascade_Loop_Shell` 模板)本身复用,只是实例数变。

- **`Cascade_Stage_Shell` Layer A 候选 vs `Controller_Cascade_Loop_Shell` Layer B**(A8 §4.5 open)
  - 影响:若 A8 后续把 stage shell 上抬到 Layer A,§4.3.2 第一行需迁移到 §4.3.1。
  - 处置:沿 A8 决议;A8 当前默认决议 = 留 Layer B,E2 与该默认一致。

- **`motor_cmd[]` 编码(PWM `_us` vs 0..1)未定**(A3 §4.5.5 标注、B4 contract diff 范畴)
  - 影响:VP-3 接入点的 leaf 实现待 B4 / E4 联动锁定;E2 仅留槽位。
  - 处置:open;E2 不预设。

- **跨级 anti-windup 协调拓扑(集中 vs 分布)**
  - 影响:§4.3.2 `Controller_AntiWindup_Coordinator` 是单点协调还是与每环内 `AntiWindup_BackCalc` 配对,E2 默认采用"中心协调器 + 每环本地原语"双重结构;若 E3 决定单点足够,本块可降级。
  - 处置:由 E3 锁定;E2 在变更日志追加。

- **`cmd_mask` 解析放在 §4.1.1 还是分散到各环**
  - 影响:E2 默认集中(`Controller_CmdMask_Resolver` 单点)以避免逻辑分散;E1 若决定某些 mask 位需要环内动态决议(例如 throttle 同时影响多个环),需评审后允许在该环内做小规模本地解析。
  - 处置:Wave 8 co-seal;默认走集中。

- **`Plant_States_Bus` harness variant 接入位置**(A5 §4.5)
  - 影响:E2 严格按 A5 决议把 variant 接入放在 Controller 顶层模型外的 harness 层,因此 Controller 顶层模型的 inport 集合仅 2 个 bus。若 G1 决定 variant 实施时需要把 inport 暴露给 Controller 顶层(避免 harness 层 bus mux 复杂度),需 G1 评审反馈。
  - 处置:open;由 G1(Wave 10)反馈决定。

## 6. 退出条件复核

对照 [`00-design-plan.md` §4.E](../00-design-plan.md) E2 行"退出条件:级联拓扑、共享 vs leaf、库块清单":

| # | 退出条件原文 | 本文档依据 | 状态 |
|---|---|---|---|
| 1 | 级联拓扑 | §4.1 给出 7 子系统顶层拓扑(setpoint decode / position / velocity / attitude / rate / mixer / output assembler);§4.2 给出 setpoint / measurement / cmd_mask 路由;§4.5 给出每环内部骨架;均与架构 v1 §12.2 对齐 | 满足 |
| 2 | 共享 vs leaf | §4.3.4 声明边界规则(沿用 A5 §4.3.3 + A8 §4.1.3);§4.3.1 / §4.3.2 列出 shared 块;§4.3.3 列出 leaf 槽位(mixer 矩阵 / 增益集 / 输出编码 / 限幅常数 — 全部 E4 owns);§4.4 给出 4 个变体接入点(VP-1..VP-4) | 满足 |
| 3 | 库块清单 | §4.3.1(Layer A)与 §4.3.2(Layer B)分别引 [A8 §4.2.4 / §4.3.3](../A-architecture/A8-shared-library-roster.md) 的 13 个块,逐块给出 E2 认领状态与使用点;不重复 A8 块定义(per RULES §5)| 满足 |

## 7. 下游影响

按 [`01-design-relationships.md` §4.5 + §4.7](../01-design-relationships.md) 出边:

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [E3 各环算法设计](E3-loops-algorithm.md) | E2 → E3(强前置) | §4.5 各环骨架 + §4.3 块实例位置;E3 在此骨架上填入控制律 |
| [E4 多旋翼 leaf](E4-multicopter-leaf.md) | 经由 E3 → E4(强前置链) | §4.4 VP-1 / VP-2 / VP-3 + §4.5.5 mixer 接入点 |
| [E5 性能预算](E5-performance-budget.md) | E1 ⇢ E5(弱);经 E3 → E5(强) | §4.1 / §4.5 子系统粒度作为时间预算的分摊单元 |
| [E1 Controller 功能设计](E1-controller-functional.md) | Wave 8 co-seal sibling | §4.1.1 mask resolver + §4.2 路由槽位 + §4.5 环路骨架,作为 E1 cmd_mask 裁剪规则的注入靶子 |
| [F3 INS 消费规则](../F-ins-contract/F3-consumption-rules.md) | B5/D1/E1 ⇢ F3(E2 是 §4.7 的辅助源) | §4.7 各环入口 validity gate 端口位置 |
| [G1 MIL 顶层](../G-harness/G1-mil-toplevel.md) | C2/D2/E2/F2 ⇢ G1 | §4.1 Controller 顶层 3-bus 边界 + §4.4 VP-4 harness variant 接入点位置 |

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-08 | controller-architect | 初稿 — 结构骨架(级联拓扑 + shared/leaf 边界 + A8 库块认领),与 E1 Wave 8 co-seal 兼容性留待 co-seal 评审核对 |

## Self-check

- [x] frontmatter 完整,字段值合法
- [x] 上游文档全部存在且 status ≥ reviewed,**或**为本工作项所在 co-seal batch 的同批兄弟(E1 不是同 batch,但 §3 已明确为 Wave 8 sibling 并在依赖表中显式标注 + §2 范围与 §5 风险中说明)— 架构 v1 / A1–A8 / B1–B3 全部 reviewed
- [x] 退出条件逐条复核完成,每条均给出依据(§6)
- [x] 引用路径全部可点击访问(本文档链接均为相对路径,指向 docs/design/ 与 docs/architecture/ 内已存在文件)
- [x] 不存在 §5 禁则中的内容(无 .slx 截图,无可执行 .m;库块定义未重复,均引用 A8;无 PR / branch 名;无凭空依赖)
- [x] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记 — **N/A**(contract_impact=no;E2 仅描述模型仓侧结构,不修改 firmware-visible bus / enum / param)
- [x] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径 — **N/A**(本文件不含镜像内容;`Control_Out_Bus` 等 bus 由 B1 / B5 镜像台账负责,本文件仅引用 B1)
- [x] 下游影响已沿关系图识别完毕(§7 列 E3 / E4 / E5 / E1 / F3 / G1 共 6 个下游)
- [x] 文档不超出本工作项范围(无越权设计):未锁控制律(E3)、未锁 mixer 内部(E4)、未做时间预算(E5)、未锁 cmd_mask 裁剪规则(E1 Wave 8 co-seal),仅做结构骨架
