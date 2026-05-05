---
work_item: A8
title: 共享库块清单与归属
upstream: ["架构 v1", "A1"]
contract_impact: no
status: reviewed
authored_at: 2026-05-05
last_reviewed_at: 2026-05-05
reviewer_verdict: pass
---

# A8 共享库块清单与归属

## 1. 目的

为 FMT-Model-2025b 仓库给出一份**横切的、跨模块的共享库块台账(roster)**:列出每一个候选共享库块的名字、所属层(Layer A 核心 / Layer B 模块共享 / Layer C 整机叶)、目录归属、消费方与版本字段。本文件作为 [C2 Plant 结构设计](../C-plant/C2-plant-structural.md)、[D2 FMS 结构设计](../D-fms/D2-fms-structural.md)、[E2 Controller 结构设计](../E-controller/E2-controller-structural.md) 三方在结构设计时**认领 / 弃权 / 协调**共享块的单一参照源,避免三个模块各自推一份不一致的共享候选清单。

## 2. 范围

**在范围:**

- Layer A(`model/shared/lib/{math,filters,guidance,control,safety,utilities}`)中跨模块通用块的**清单与归属**
- Layer B(`model/<module>/shared/`)中**单个模块内**跨整机复用的架构骨架块的清单与归属
- 每个块的:块名、所属层、所属子目录、消费者(C/D/E/F/G 区)、版本字段约定、备注
- 命名/层级二义时的**冲突解决规则**(默认上抬到更高层)
- "open" 区域:目前归属未决的候选块,标注理由

**不在范围(由其他工作项处理):**

- 共享库块**内部实现**(由 C2/D2/E2 在自己的子结构设计中,或由后续实现阶段处理)
- 库块的**精确命名规则**(由 [A2 命名与目录约定设计](A2-naming-conventions.md) 处理 — A8 仅给"工作名" / placeholder,A2 落定最终命名)
- 各机型 leaf 块清单(由 [C3 Plant 多旋翼 leaf](../C-plant/C3-multicopter-leaf.md)、[D5 FMS 多旋翼 leaf](../D-fms/D5-multicopter-leaf.md)、[E4 Controller 多旋翼 leaf](../E-controller/E4-multicopter-leaf.md) 等处理 — A8 仅以 `<vehicle>/leaf` 占位)
- 库块的版本号**值**(块版本由其归属工作项在落地时填写;本文件只规定格式约束)
- 库块**接口字段**(若涉及契约 bus,由 B1/B2 处理;库块自身的输入/输出端口由 C2/D2/E2 在结构设计中给出)

## 3. 依赖

| 上游 | 引用位置 | 用途 |
|---|---|---|
| 架构 v1 §6 仓库树 | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | `model/shared/lib/{math,filters,guidance,control,safety,utilities}` 与 `model/<module>/shared/` 的物理归属位置 |
| 架构 v1 §10 Shared-vs-vehicle layering | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | Layer A/B/C 三层划分;Layer A 通用块清单(deadzone, rate limiter, LPF, mission decoder, quaternion utils, common safety check);Layer B 模块共享块清单;§10.2 修订声明"无 INS 层" |
| 架构 v1 §17 Risk 3 Over-abstraction | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | 防止过早抽象的依据 — 本文件中"未被任一现有消费者使用的候选块"原则上不进入 Layer A |
| A1 §4.2 §6/§9.2/§10/§12 行 | [A1 v1 评审](A1-v1-review.md) | A1 §4.2 §6 行(`model/shared/lib/{math,filters,...}` 各子目录的入选库块清单未定,Closed by C2/D2/E2);A1 §4.2 §9.2.4(内部 bus 模块前缀化);A1 §4.2 §10.1/§10.2/§10.3(safety/loop shells/leaf 反例豁免);A1 §4.4 推荐设立 A6 *(proposed)*(横切共享库块汇总)— 本工作项即为该推荐项,经 2026-05-05 审计扩计划时正式编号为 A8 |

注:A8 是 [00-design-plan §4.A](../00-design-plan.md) 中 8 个 A 区工作项的第 8 项,2026-05-05 审计扩计划时新增,以闭合 A1 §4.4 *(proposed A6)* 的"共享库块清单与归属总表"建议(因 A6 已被 init/reset 状态契约占用,本项编号为 A8)。

A8 的下游 C2/D2/E2 在 Wave 7 启动,因此 A8 必须在 Wave 2 完成(与 A2/A4/A5/A6/A7 同 Wave 并行)。A8 与同 Wave 的 A2 之间存在弱协同:A2 命名规则细化时若需要"块命名约定",可参考本文件枚举的块名作为输入;反之,本文件中的块名是**工作名**,以 A2 最终命名为准。

## 4. 设计内容

### 4.1 设计原则

#### 4.1.1 三层划分(对应架构 v1 §10)

- **Layer A(shared core)**:通用、与机型无关、跨 ≥ 2 个模块复用的块。物理位置:`model/shared/lib/<sub>/`,其中 `<sub>` ∈ `{math, filters, guidance, control, safety, utilities}`。
- **Layer B(module-shared architecture)**:仅服务**单一模块**(Plant 或 FMS 或 Controller),但跨该模块的所有机型 leaf 复用的块或子系统骨架。物理位置:`model/<module>/shared/`。
- **Layer C(vehicle leaf)**:整机相关逻辑,仅在某一机型(multicopter / fixwing / vtol / boat / car / submarine)内使用。物理位置:`model/<module>/vehicles/<vehicle>/`。**A8 不枚举 Layer C 的具体叶块**,仅以 `<vehicle>/leaf` 占位,具体清单留给 C3/D5/E4 等机型 leaf 工作项。

按架构 v1 §10.2 修订(2026-05-05),**Layer B 中不存在 INS 层**;模型仓侧唯一与 INS 相关的资产是 `model/harness/ins_stub/`(MIL-only 验证夹具,**不属于** Layer A/B/C 任意层)— 由 [F1 ins_stub 功能设计](../F-ins-contract/F1-ins-stub-functional.md)、[F2 结构设计](../F-ins-contract/F2-ins-stub-structural.md) 单独处理,本文件不收录。

#### 4.1.2 入选准则

某块进入本台账的条件:

1. **预期被 ≥ 1 个 C/D/E/F 区下游消费**(单消费者也可入选 Layer B,但 Layer A 至少需 ≥ 2 个跨模块消费者,以避免架构 v1 §17 Risk 3 over-abstraction);
2. **块本身无机型相关参数**(机型相关常量留 leaf);
3. **可被 Simulink Library 块或 Subsystem Reference 形式承载**;
4. **接口稳定**:块的端口签名期望多机型或多模块复用而不需要每次特化。

凡不满足上述任一条的块,留 Layer C 或独立模块内部块,不进入本台账。

#### 4.1.3 冲突解决规则(同名/同语义块的归层)

当某块**既可入 Layer A 也可入 Layer B**(例如某 LPF 当前仅 Controller 使用,但语义上是通用滤波器):

- **默认决议:上抬到更高(更共享)层**(Layer A > Layer B > Layer C),使下游可以无障碍接入;
- 例外情形(必须明示理由):
  - 块的实现**强耦合于该模块的内部信号语义**(例:Controller 内的 `cmd_mask` 解释器),不能脱离上下文使用 → 留 Layer B;
  - 块的实现**包含机型相关常量或假设**(例:multicopter 专用混控)→ 留 Layer C;
  - 块的接口预计在短期内频繁变化(尚未稳定),**强提早共享会引发频繁全仓回归** → 暂留 Layer B,标 "open: candidate to lift",并在变更日志中记录。

冲突解决的决议过程在每个块的 "Notes" 列说明。

#### 4.1.4 命名与版本字段约定(指针)

- **命名**:本文件中给出的块名为"工作名"(working name),格式为 PascalCase 或下划线分隔的英文短语,仅供 C2/D2/E2 在结构设计中识别引用。**最终命名规则**(包括前缀、大小写、子目录命名)由 [A2 命名与目录约定设计](A2-naming-conventions.md) 落定;A2 完稿后,本文件可由变更日志触发同步重命名。
- **版本字段**:每个共享库块**必须**携带一个版本字符串字段,可在 Simulink 块的 `Description` / `MaskInitialization` 或库回调(InitFcn / PreLoadFcn)中读出,用于:
  - 库内升级时下游识别变更(配合 [B4 契约 diff 策略](../B-contracts/B4-contract-diff.md) 中的库块版本对照,但**库块版本不属于 firmware 契约**,故不入 contract diff 报告;仅模型仓侧自检使用);
  - 跨模块协同冻结(co-seal batch 时各模块 freeze 同一版本快照)。
- **版本格式**:`vN`(整数语义版本)或 `YYYY-MM-DD`(日期串)。具体格式由 A2 在命名规则一并落定;本文件视为 **TBD pending A2**,同时给出最低约束:格式必须可被 Simulink 的 `get_param` 或自定义脚本读取,且必须可与自动 contract-diff 工具(I3)区分(库块版本不应被 I3 误判为 firmware 契约变更)。

### 4.2 Layer A — shared core 库块台账

物理目录:`model/shared/lib/<sub>/`(架构 v1 §6)。下表覆盖架构 v1 §10.1 给出的核心通用块,加上 A1 §4.2 / 内部 bus 前缀清单 / 共享数据通路扫读派生的衍生项。

#### 4.2.1 `math/` — 通用数学块

| Block name | Owner area | Owning subdir | Consumers (areas) | Version field | Notes |
|---|---|---|---|---|---|
| `Quaternion_Mul` | Layer A | `model/shared/lib/math/` | C(plant 姿态积分)、D(FMS 姿态参考插值,若使用)、E(controller 姿态环误差) | TBD pending A2 (default `vN`) | 架构 v1 §10.1 "generic quaternion utilities";多个模块都需要四元数乘法 |
| `Quaternion_Conj` | Layer A | `model/shared/lib/math/` | C, D, E | TBD pending A2 | 与上同,提供共轭 |
| `Quaternion_Normalize` | Layer A | `model/shared/lib/math/` | C, D, E | TBD pending A2 | 归一化以避免数值漂移;C4 数值设计可决定具体重整频率 |
| `Quat_To_Rotation_Matrix` | Layer A | `model/shared/lib/math/` | C(plant 坐标系变换)、E(controller 误差解算) | TBD pending A2 | 架构 v1 §10.1 "generic quaternion/rotation utilities" |
| `Rotation_Matrix_To_Quat` | Layer A | `model/shared/lib/math/` | C, E | TBD pending A2 | 反向工具 |
| `Euler_To_Quat` | Layer A | `model/shared/lib/math/` | D(姿态 setpoint 来源若为欧拉)、E、test fixture | TBD pending A2 | 仅在边界使用,主链路用四元数 |
| `Quat_To_Euler` | Layer A | `model/shared/lib/math/` | G(harness 日志可读化)、H(验证可读化) | TBD pending A2 | 主要用于日志/验证,运行时不在主路径 |
| `Vec3_Cross` | Layer A | `model/shared/lib/math/` | C, E | TBD pending A2 | 三维向量叉积;角动量 / 离心校正 |
| `Vec3_Dot` | Layer A | `model/shared/lib/math/` | C, E | TBD pending A2 | 三维向量点积 |
| `Frame_Transform_NED_Body` | Layer A | `model/shared/lib/math/` | C, E | TBD pending A2 | NED↔body 坐标系变换;A2 单位/坐标系约定落定后,本块的 frame 标识由 A2 给出 |

#### 4.2.2 `filters/` — 通用滤波器

| Block name | Owner area | Owning subdir | Consumers (areas) | Version field | Notes |
|---|---|---|---|---|---|
| `LowPass_Filter_1st` | Layer A | `model/shared/lib/filters/` | C(plant 传感器合成,可选)、D(FMS 命令整形输入平滑)、E(controller 测量预滤波) | TBD pending A2 | 架构 v1 §10.1 "low-pass filters" |
| `HighPass_Filter_1st` | Layer A | `model/shared/lib/filters/` | E(controller D 项预处理,若不内嵌于 PID 块) | TBD pending A2 | 通常作为 controller 内的 D-term 整形;若内嵌在 PID 中,可降级为 Layer B,但默认上抬 |
| `Notch_Filter` | Layer A | `model/shared/lib/filters/` | E(角速率环抑振)、可选 C(传感器后端) | TBD pending A2 | 多旋翼角速率环常需,候选 |
| `Moving_Average` | Layer A | `model/shared/lib/filters/` | C, D, E, G(日志平滑) | TBD pending A2 | 通用 |
| `Rate_Limiter` | Layer A | `model/shared/lib/filters/` | D(命令整形)、E(参考输入限速) | TBD pending A2 | 架构 v1 §10.1 "rate limiters";Simulink 内置块可包装为统一接口 |
| `Slew_Limiter` | Layer A | `model/shared/lib/filters/` | D, E | TBD pending A2 | 与 rate limiter 区分:slew 含上下不对称限速 |
| `Deadzone` | Layer A | `model/shared/lib/filters/` | D(pilot stick 中心死区)、E(控制误差死区) | TBD pending A2 | 架构 v1 §10.1 "deadzones" |
| `Saturation_Symmetric` | Layer A | `model/shared/lib/filters/` | C, D, E | TBD pending A2 | 对称限幅 |
| `Saturation_Asymmetric` | Layer A | `model/shared/lib/filters/` | C, D, E | TBD pending A2 | 非对称限幅;upper/lower 独立 |

注:`Rate_Limiter`/`Slew_Limiter`/`Deadzone`/`Saturation_*` 在 Simulink 内置块基础上做"统一接口包装",目的是为版本字段、单位 metadata、坐标系标注提供一致的 mask 化封装(参见 A2)。

#### 4.2.3 `guidance/` — 通用制导/任务辅助块

| Block name | Owner area | Owning subdir | Consumers (areas) | Version field | Notes |
|---|---|---|---|---|---|
| `Mission_Item_Decoder` | Layer A | `model/shared/lib/guidance/` | D(FMS Mission/Auto Manager,见 §12.1.4) | TBD pending A2 | 架构 v1 §10.1 "generic mission item decoding helpers";闭环切片范围由 [D1 FMS 功能设计](../D-fms/D1-fms-functional.md) 决定 |
| `Waypoint_Distance_Calc` | Layer A | `model/shared/lib/guidance/` | D(到点判定) | TBD pending A2 | 距离/方位辅助;若机型差异大可下沉,默认上抬 |
| `Geofence_Check` | Layer A | `model/shared/lib/guidance/` | D(safety 监视,见 §12.1.3 failsafe) | TBD pending A2 | "open: 候选 — 是否落 Layer A 取决于 D1 是否将 geofence 列入 Phase 1 切片";若 D1 推迟 geofence,本块可降为 Layer B(FMS shared) |

#### 4.2.4 `control/` — 通用控制原语

| Block name | Owner area | Owning subdir | Consumers (areas) | Version field | Notes |
|---|---|---|---|---|---|
| `PID_Block` | Layer A | `model/shared/lib/control/` | E(各级环) | TBD pending A2 | 通用 PID 单元(含抗 windup hook、D-term LPF hook、前馈输入);具体 anti-windup 算法由 [E3 Controller 各环算法设计](../E-controller/E3-loops-algorithm.md) 决定 |
| `AntiWindup_BackCalc` | Layer A | `model/shared/lib/control/` | E | TBD pending A2 | 反算式抗 windup;如 E3 选用 conditional integration,本块可弃用或并存 |
| `FeedForward_Composer` | Layer A | `model/shared/lib/control/` | E | TBD pending A2 | 加性前馈合成,提供统一注入点 |
| `Cascade_Stage_Shell` *(open)* | Layer A or Layer B | `model/shared/lib/control/` 或 `model/controller/shared/` | E | TBD pending A2 | "open: 候选 — 与 §4.3.3 `Controller_Cascade_Loop_Shell` 重叠,二选一";架构 v1 §10.2 把 "Controller shared cascade loop shells" 列在 Layer B,本文件**默认遵循 v1**,留 Layer B(见 §4.3.3),Layer A 不重复列入 |

#### 4.2.5 `safety/` — 通用安全检查原语

| Block name | Owner area | Owning subdir | Consumers (areas) | Version field | Notes |
|---|---|---|---|---|---|
| `Threshold_Latch` | Layer A | `model/shared/lib/safety/` | D(failsafe 阈值持锁)、E(过限标志) | TBD pending A2 | 阈值越界 + 持锁;与 hysteresis 配套 |
| `Hysteresis_Comparator` | Layer A | `model/shared/lib/safety/` | D, E | TBD pending A2 | 双阈值比较 |
| `Validity_AND` | Layer A | `model/shared/lib/safety/` | D(消费 INS validity)、E(同) | TBD pending A2 | 多路 validity 位与;接口与 [F3 INS_Out_Bus 消费规则](../F-ins-contract/F3-consumption-rules.md) 中的 validity 字段对齐 |
| `Validity_OR` | Layer A | `model/shared/lib/safety/` | D, E | TBD pending A2 | 多路或 |
| `Stale_Detector` | Layer A | `model/shared/lib/safety/` | D(消息超时)、E(同) | TBD pending A2 | 消息时间戳超时检测;时间戳约定由 [A7 时间与时间戳约定](A7-time-conventions.md) 给出 |
| `Range_Check` | Layer A | `model/shared/lib/safety/` | D, E, C(plant 输出范围健全性) | TBD pending A2 | 上下界健全性 |

注:架构 v1 §10.1 列出的 "common safety checks" 在本文件中拆为上述若干**原语**。**FMS 安全策略骨架**(组合多原语形成的 failsafe 决策树)归 Layer B(`fms/shared/`,见 §4.3.2),不在 Layer A。这一拆分对齐 A1 §4.2 §10.1 "common safety checks 是否包括 FMS 安全策略骨架" 的 open 决议(默认 No,但原语在 Layer A)。

#### 4.2.6 `utilities/` — 通用辅助块

| Block name | Owner area | Owning subdir | Consumers (areas) | Version field | Notes |
|---|---|---|---|---|---|
| `Bus_Selector_Wrapped` | Layer A | `model/shared/lib/utilities/` | C, D, E, F, G | TBD pending A2 | Simulink Bus Selector 的统一封装,提供 reset 默认值 hook(与 [A6 Init/Reset 状态契约](A6-init-reset-contract.md) 对接) |
| `Enum_Constant` | Layer A | `model/shared/lib/utilities/` | D, E, F | TBD pending A2 | 枚举常量包装,值来源由 [B2 Enum 清单](../B-contracts/B2-enum-inventory.md) 给出 |
| `Unit_Cast` | Layer A | `model/shared/lib/utilities/` | C, D, E, G | TBD pending A2 | 单位换算辅助(rad↔deg, m↔mm, ...);单位约定由 A2 给出 |
| `Fixed_Step_Counter` | Layer A | `model/shared/lib/utilities/` | C, D, E, G | TBD pending A2 | 周期计数,与 reset 行为对接 |
| `Reset_Composer` | Layer A | `model/shared/lib/utilities/` | C, D, E | TBD pending A2 | 多路 reset 信号合成;与 A6 init/reset 契约对接 |

#### 4.2.7 Layer A 计数

Layer A 共列出 **37** 个块(math 10 + filters 9 + guidance 3 + control 4 + safety 6 + utilities 5 — 其中 `Cascade_Stage_Shell` 标 *(open)* 但**默认决议为不入 Layer A**;`Geofence_Check` 标 *(open)* 但默认 Layer A)。

实际下游可使用的稳定 Layer A 块数 ≥ 36(扣除 1 个 *(open)* 的待决项 `Cascade_Stage_Shell`)。

### 4.3 Layer B — module-shared architecture 库块台账

物理目录:`model/<module>/shared/`(架构 v1 §6)。**架构 v1 §10.2 修订(2026-05-05)显式声明无 INS 层**;故仅 plant / fms / controller 三个模块。

#### 4.3.1 `model/plant/shared/` — Plant 共享架构块

| Block name | Owner area | Owning subdir | Consumers (areas) | Version field | Notes |
|---|---|---|---|---|---|
| `Plant_Sensor_Synthesis_Shell` | Layer B (Plant) | `model/plant/shared/` | C(各机型 plant leaf)、F(ins_stub 输入也读 Plant_States_Bus,共享同一套传感器合成接口) | TBD pending A2 | 架构 v1 §10.2 "Plant shared sensor synthesis blocks";子结构含 IMU/GPS/baro/mag 合成框架,机型常量在 leaf 注入 |
| `Rigid_Body_Dynamics_Shell` | Layer B (Plant) | `model/plant/shared/` | C(各机型 leaf 复用 6-DOF 刚体框架) | TBD pending A2 | 6-DOF 刚体动力学骨架;质量/惯量在 leaf 注入 |
| `Aero_Force_Frame_Shell` *(open)* | Layer B (Plant) | `model/plant/shared/` | C(若 fixwing/vtol 加入,则共享气动框架) | TBD pending A2 | "open: 候选 — 当前 00-design-plan §5 排除 multicopter 以外机型;若 Phase 2 接入 fixwing,本块需提级";暂留 Layer B,候选 |
| `Disturbance_Injector_Shell` | Layer B (Plant) | `model/plant/shared/` | C, G(harness 故障注入), H4(故障目录) | TBD pending A2 | 通用扰动注入框架(风、气压扰动、传感器噪声预留接口);具体扰动场景由 [H4 故障注入目录](../H-verification/H4-fault-catalog.md) 决定 |
| `Plant_Reset_Hooks_Shell` | Layer B (Plant) | `model/plant/shared/` | C | TBD pending A2 | reset 入口统一封装,与 [A6 Init/Reset 状态契约](A6-init-reset-contract.md) 对接 |

#### 4.3.2 `model/fms/shared/` — FMS 共享架构块

| Block name | Owner area | Owning subdir | Consumers (areas) | Version field | Notes |
|---|---|---|---|---|---|
| `FMS_Mode_Manager_Framework` | Layer B (FMS) | `model/fms/shared/` | D(各机型 FMS leaf)、D3 详细设计填具体状态 | TBD pending A2 | 架构 v1 §10.2 "FMS shared mode manager framework";Stateflow chart 骨架(状态名/事件接口/守卫挂钩),状态数与转换由 [D3 FMS Mode Manager](../D-fms/D3-mode-manager.md) 决定 |
| `FMS_Safety_Monitor_Framework` | Layer B (FMS) | `model/fms/shared/` | D | TBD pending A2 | 架构 v1 §10.2 "FMS shared safety monitor";failsafe 决策树骨架(组合 Layer A `Threshold_Latch`/`Stale_Detector`/`Validity_AND` 等原语);具体阈值与触发动作由 [D1](../D-fms/D1-fms-functional.md) 决定 |
| `FMS_Mission_Decoder_Shell` | Layer B (FMS) | `model/fms/shared/` | D | TBD pending A2 | Mission/Auto Manager 框架(消费 Layer A `Mission_Item_Decoder`);闭环切片需要的子集由 D1/D5 决定 |
| `FMS_Source_Selector_Shell` | Layer B (FMS) | `model/fms/shared/` | D | TBD pending A2 | 命令源(pilot / ground / mission)选择器骨架;与 [D6 cmd_mask 语义](../D-fms/D6-fms-controller-interface.md) 对接 |
| `FMS_Command_Shaper_Shell` | Layer B (FMS) | `model/fms/shared/` | D | TBD pending A2 | 命令整形骨架(组合 Layer A `Rate_Limiter`/`Slew_Limiter`/`Deadzone`);具体 mode-cmd 映射与限值由 [D4](../D-fms/D4-command-shaper.md)/[D5](../D-fms/D5-multicopter-leaf.md) 决定 |
| `FMS_Output_Assembler_Shell` | Layer B (FMS) | `model/fms/shared/` | D | TBD pending A2 | `FMS_Out_Bus` 装配骨架;字段由 [B1 Bus 清单](../B-contracts/B1-bus-inventory.md) 给出 |
| `FMS_Reset_Hooks_Shell` | Layer B (FMS) | `model/fms/shared/` | D | TBD pending A2 | 与 [A6](A6-init-reset-contract.md) 对接 |

#### 4.3.3 `model/controller/shared/` — Controller 共享架构块

| Block name | Owner area | Owning subdir | Consumers (areas) | Version field | Notes |
|---|---|---|---|---|---|
| `Controller_Cascade_Loop_Shell` | Layer B (Controller) | `model/controller/shared/` | E(各机型 controller leaf) | TBD pending A2 | 架构 v1 §10.2 "Controller shared cascade loop shells";单级 cascade(setpoint → error → law → output)骨架;具体 law 在内部插入 Layer A `PID_Block`;与 §4.2.4 `Cascade_Stage_Shell` *(open)* 二选一,**本文件决议留 Layer B**(块**强耦合于 controller 的 cmd_mask 解释和 anti-windup 协调**) |
| `Controller_CmdMask_Resolver` | Layer B (Controller) | `model/controller/shared/` | E | TBD pending A2 | 解释 [D6 cmd_mask 语义](../D-fms/D6-fms-controller-interface.md) 给出的位掩码 → 启用的环路;裁剪规则由 [E1](../E-controller/E1-controller-functional.md) 在 D6+E1 co-seal batch 中给出 |
| `Controller_AntiWindup_Coordinator` | Layer B (Controller) | `model/controller/shared/` | E | TBD pending A2 | 跨级 anti-windup 协调(Layer A 的 `AntiWindup_BackCalc` 原语提供单点功能;本块负责级联间协调) |
| `Controller_Mixer_Frame_Shell` | Layer B (Controller) | `model/controller/shared/` | E(各机型 mixer leaf 实现具体分配矩阵) | TBD pending A2 | 混控框架骨架(输入:虚拟控制力/力矩;输出:执行器命令向量);具体分配矩阵在 leaf(C3/E4) |
| `Controller_Output_Saturator_Shell` | Layer B (Controller) | `model/controller/shared/` | E | TBD pending A2 | 执行器输出限幅骨架(组合 Layer A `Saturation_*`);限值在 leaf 注入 |
| `Controller_Reset_Hooks_Shell` | Layer B (Controller) | `model/controller/shared/` | E | TBD pending A2 | 与 [A6](A6-init-reset-contract.md) 对接 |

#### 4.3.4 Layer B 计数

Layer B 共列出 **18** 个块(plant 5 + fms 7 + controller 6),其中 `Aero_Force_Frame_Shell` 标 *(open)* 候选,稳定 Layer B 块数 ≥ 17。

### 4.4 Layer C — 整机叶占位

物理目录:`model/<module>/vehicles/<vehicle>/`(架构 v1 §6)。**A8 不枚举具体叶块**;以下仅作占位说明:

| Placeholder | Owner area | Owning subdir | Consumers (areas) | Version field | Notes |
|---|---|---|---|---|---|
| `<vehicle>/leaf` | Layer C | `model/<module>/vehicles/<vehicle>/` | 各机型自身 | TBD pending A2 | 具体清单由 [C3](../C-plant/C3-multicopter-leaf.md)、[D5](../D-fms/D5-multicopter-leaf.md)、[E4](../E-controller/E4-multicopter-leaf.md) 等机型 leaf 工作项给出 |

按 [00-design-plan §5](../00-design-plan.md),设计阶段仅细化多旋翼 leaf;其他机型(fixwing/vtol/boat/car/submarine)的 leaf 设计推迟。

### 4.5 Open / TBD 项专列

| 项 | 类别 | 当前状态 | 处置 |
|---|---|---|---|
| `Cascade_Stage_Shell` (Layer A 候选) | open | 与 §4.3.3 `Controller_Cascade_Loop_Shell` 重叠 | 默认决议:**留 Layer B**,Layer A 不重复列入;若 E2 决议拆分为更通用的 stage 原语,触发本文件变更日志 |
| `Geofence_Check` (Layer A 候选) | open | 是否入 Layer A 取决于 D1 是否将 geofence 列入 Phase 1 闭环切片 | 默认决议:**留 Layer A**;若 D1 推迟 geofence,本块可降为 Layer B |
| `Aero_Force_Frame_Shell` (Layer B candidate) | open | 当前 00-design-plan §5 排除多旋翼以外机型 | 默认决议:**留 Layer B 候选**,设计阶段不实现;Phase 2 接入 fixwing 时再激活 |
| 所有块的 **最终命名** | TBD pending A2 | 工作名为 placeholder | A2 完稿后由变更日志触发同步;在 A2 落定前,本文件块名可被引用作为工作名 |
| 所有块的 **版本字段格式**(`vN` vs `YYYY-MM-DD`) | TBD pending A2 | 仅约定"必须存在 + 必须可读" | A2 完稿后填入;本文件定稿不阻塞 A2 |
| `model/shared/data/` 用途 | open | A1 §4.2 §6 行;C3+D5+E4 决定是否使用 | A8 不强制采用,留给 leaf 工作项决定 |

### 4.6 与下游(C2/D2/E2)的认领协议

C2/D2/E2 在自己的结构设计中,使用本文件的台账时按以下流程:

1. **认领(claim)**:在结构设计文档的"共享库块清单"小节中,**逐行引用**本文件 §4.2/§4.3 的相关行(例:"消费 `model/shared/lib/filters/Rate_Limiter`"),无需复述本文件的列内容。
2. **弃权(disclaim)**:若某行台账中的块未被该模块消费,在结构设计的开头/结尾说明"未使用本台账中的 `<块名>`"或在 §下游影响中列出未引用清单。
3. **新增(propose addition)**:若结构设计中发现需要某共享块但本文件未列入,**通过本文件的变更日志追加**(走 [RULES §10](../RULES.md)),并通知 A8 reviewer 与同 Wave 兄弟。
4. **下沉(propose demotion)**:若某 Layer A 块在该模块设计中被发现不再通用,提议下沉到 Layer B 或 Layer C,经评审后更新本文件。
5. **上抬(propose promotion)**:若发现本文件 Layer B 中的块在多个模块都需要,提议上抬到 Layer A,经评审后更新本文件。

冲突时遵循 §4.1.3 默认决议(上抬到更高层),例外需说明。

### 4.7 自检清单(给 C2/D2/E2 reviewer 用)

reviewer 在评审 C2/D2/E2 时,可以用本文件做反向核对:

- [ ] C2 是否认领或弃权了 `model/plant/shared/` 中所有 Layer B 块?
- [ ] D2 是否认领或弃权了 `model/fms/shared/` 中所有 Layer B 块?
- [ ] E2 是否认领或弃权了 `model/controller/shared/` 中所有 Layer B 块?
- [ ] C2/D2/E2 是否合计认领了 Layer A 中至少 ≥ 1 个 math/filters/safety 子目录的块?
- [ ] 任一模块新增的"共享候选"是否走了 §4.6 的新增流程,在 A8 中登记?

## 5. 已知风险与悬而未决问题

- **A2 命名规则未定即引用块名**
  - 影响:本文件中的所有块名为工作名,A2 完稿后可能整体重命名;C2/D2/E2 在 Wave 7 已引用本文件
  - 处置:本文件 §4.1.4 明示"工作名 + TBD pending A2";A2 完稿时同 Wave 协同对(A2 与 A8 同 Wave 2)使用 RULES §10 变更日志机制集中重命名;C2/D2/E2 通过引用本文件路径保持稳定指针,具体块名通过本文件的变更日志感知重命名

- **过早抽象风险(架构 v1 §17 Risk 3)**
  - 影响:Layer A 列出 37 个候选块(其中 1 项 *open*),若实际只有少数被消费,会引入不必要的库维护负担
  - 处置:§4.1.2 给出入选准则(Layer A 至少 ≥ 2 跨模块消费);§4.6 提供下沉机制;在实现阶段第一个 milestone(Phase 1 多旋翼闭环切片)结束时,reviewer 应对未被实际消费的 Layer A 块做一次裁剪复审

- **Layer A 与 Layer B 边界二义**
  - 影响:`Cascade_Stage_Shell` vs `Controller_Cascade_Loop_Shell`、`Geofence_Check` 的层级裁决可能在 D1/E2 完稿时反转
  - 处置:§4.5 显式列为 open;§4.1.3 给出默认决议规则(上抬);任何反转走变更日志

- **`safety/` 中安全原语 vs FMS 安全策略骨架的边界**
  - 影响:架构 v1 §10.1 文本"common safety checks"在 A1 §4.2 §10.1 行被标 open(是否含 FMS 安全策略骨架)
  - 处置:本文件 §4.2.5 明确把**原语**(`Threshold_Latch`/`Hysteresis_Comparator`/`Stale_Detector`/...)留 Layer A;**FMS 安全策略骨架**(`FMS_Safety_Monitor_Framework`)归 Layer B(§4.3.2);此决议封闭 A1 §4.2 §10.1 的 open 标记

- **库块版本字段与 firmware contract diff 的混淆风险**
  - 影响:I3 contract diff 工具(由 [B4 契约 diff 策略](../B-contracts/B4-contract-diff.md) 与 [I3 契约 diff 脚本设计](../I-tooling/I3-contract-diff.md) 实现)若把库块版本误判为 firmware 契约变更,会引发误报
  - 处置:§4.1.4 明示"库块版本不属于 firmware 契约,不入 contract diff 报告";后续 B4/I3 设计时需在白名单中排除库块版本字段;**本风险登记给 B4/I3 的 reviewer**

- **`model/shared/data/` 目录未使用**
  - 影响:架构 v1 §6 树中存在 `model/shared/data/`,但本文件未为其指派任何块
  - 处置:留给 C3/D5/E4 在使用时填入;A8 不强行规定;A1 §4.2 §6 行已记录由 C3+D5+E4 闭合

## 6. 退出条件复核

A8 在 [`00-design-plan.md`](../00-design-plan.md) §4.A 中的退出条件原文:

> 跨 C2/D2/E2 的共享库块台账(块名、所有者、消费者、版本字段);A1 已识别的归属空白点全部闭合

逐条复核:

| # | 退出条件分解 | 本文档依据 | 状态 |
|---|---|---|---|
| 1 | "跨 C2/D2/E2 的共享库块台账" — 形式上是一份台账 | §4.2(Layer A 37 块,6 子目录表)+ §4.3(Layer B 18 块,3 模块表)+ §4.4(Layer C 占位行) | 满足 |
| 2 | 包含"块名" | §4.2/§4.3 表中 "Block name" 列;Layer A 36 + 1 open;Layer B 17 + 1 open | 满足 |
| 3 | 包含"所有者(Owner area)" | §4.2/§4.3 表中 "Owner area" 列;Layer A / Layer B (Plant|FMS|Controller) | 满足 |
| 4 | 包含"消费者(Consumers)" | §4.2/§4.3 表中 "Consumers (areas)" 列;以 C/D/E/F/G/H 区域字母标注 | 满足 |
| 5 | 包含"版本字段(Version field)" | §4.2/§4.3 表中 "Version field" 列;格式 TBD pending A2;§4.1.4 给出最低约束 | 满足(格式 TBD,但版本字段存在性已约束) |
| 6 | "A1 已识别的归属空白点全部闭合" — A1 §4.2 §6/§9.2/§10/§12 的 shared lib 相关 open 行 | §3 依赖表中具体引用每条 A1 行;§4.2.5 / §4.3.2 显式封闭 A1 §4.2 §10.1 行;§4.1.3 冲突解决规则封闭 A1 §4.2 §10.2/§10.3 行;§4.1.4 命名/版本指针封闭 A1 §4.2 §6 行(Closed by C2+D2+E2 加 A8 总账) | 满足 |
| 7 | 隐含:格式可被下游(C2/D2/E2)无歧义引用 | §4.6 给出认领/弃权/新增/下沉/上抬五种交互流程;§4.7 给 reviewer 反向核对清单 | 满足 |

退出条件满足;状态可由 reviewer 升级到 reviewed。

## 7. 下游影响

按 [`01-design-relationships.md §4.10`](../01-design-relationships.md):

```text
A1 → A8
A8 ⇢ C2, D2, E2
```

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [C2 Plant 结构设计](../C-plant/C2-plant-structural.md) *(未启动,Wave 7)* | ⇢ | §4.2 Layer A 通用块台账;§4.3.1 `model/plant/shared/` 5 个块的 Layer B 候选;§4.6 认领协议 |
| [D2 FMS 结构设计](../D-fms/D2-fms-structural.md) *(未启动,Wave 7)* | ⇢ | §4.2 Layer A 通用块台账;§4.3.2 `model/fms/shared/` 7 个块的 Layer B 候选;§4.6 认领协议 |
| [E2 Controller 结构设计](../E-controller/E2-controller-structural.md) *(未启动,Wave 7)* | ⇢ | §4.2 Layer A 通用块台账;§4.3.3 `model/controller/shared/` 6 个块的 Layer B 候选;§4.6 认领协议 |

间接影响(通过 C2/D2/E2 传递):

- **C3/D5/E4 多旋翼 leaf**:间接消费本文件 Layer A 块;C3 经 C2 转引,D5 经 D2 + D3+D4,E4 经 E2 + E3。
- **F3 INS_Out_Bus 消费规则**:间接通过 §4.2.5 的 `Validity_AND`/`Stale_Detector` 等原语共享 validity 处理约定。
- **G 系列**:`Disturbance_Injector_Shell`(Layer B Plant)与 G 区 harness 故障注入接口对接;G3 日志使用 Layer A `Quat_To_Euler` / `Unit_Cast` 等。
- **A2 命名规则**:与 A8 同 Wave 协同;A8 给 A2 提供工作名清单,A2 落定后回传规范命名。

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-05 | A8 author | 初稿 |

## Self-check

- [x] frontmatter 完整,字段值合法
- [x] 上游文档全部存在且 status ≥ reviewed(架构 v1 已发布;A1 status=reviewed verdict=pass,见 INDEX)
- [x] 退出条件逐条复核完成,每条均给出依据
- [x] 引用路径全部可点击访问
- [x] 不存在 RULES §5 禁则中的内容(无 .slx 截图、无可执行 .m、无重复 firmware 字段表、无 PR/branch 名)
- [x] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记(本文件 contract_impact=no — 库块版本字段不属于 firmware 契约,见 §4.1.4 与 §5 风险登记;N/A)
- [x] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径(本文件不镜像 firmware 任何契约;N/A)
- [x] 下游影响已沿关系图识别完毕(C2/D2/E2 直接;C3/D5/E4/F3/G 系列间接)
- [x] 文档不超出本工作项范围(无越权设计;命名细节让给 A2,叶清单让给 C3/D5/E4,内部实现让给 C2/D2/E2)
