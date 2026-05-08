---
work_item: F1
title: ins_stub 功能设计
upstream: ["架构 v1", "A1", "A3", "A4", "A5", "A6", "A7", "A8", "B1", "B2", "B5", "C1"]
contract_impact: no
status: draft
authored_at: 2026-05-08
last_reviewed_at:
reviewer_verdict: none
---

# F1 ins_stub 功能设计(MIL-only INS 替身的功能契约)

## 1. 目的

为 MIL/SIL 仿真闭环定义 **harness-only** 的 `ins_stub` 的**功能行为契约**:它以 Plant 输出的 `Plant_States_Bus`(truth)为唯一输入,产出一个**字段级符合 [B5](../B-contracts/B5-ins-bus-mirror.md) firmware-canonical mirror artifact** 的 `INS_Out_Bus`,以替代真实 firmware 估计器在仿真路径下闭环;并为下游故障 / 噪声 / 偏差 / 延迟场景暴露**可参数化的 knob 集合**(per 审计 [F-04](../_audit/2026-05-05-plan-audit.md))。本文件**仅**约束理想版与带扰版的输出特性、knob shape 与 init/reset 行为;**不**做块级结构(留 [F2](F2-ins-stub-structural.md)),**不**做 FMS / Controller 消费规则(留 [F3](F3-consumption-rules.md))。

## 2. 范围

**在范围:**

- §4.1 ins_stub 的目的、boundary(harness-only)、变体集合(ideal / noisy-biased-delayed)、cadence(10 ms = 100 Hz)
- §4.2 **理想版**(ideal)输出特性:identity transform from `Plant_States_Bus` → `INS_Out_Bus`;validity 立即 valid;`INS_Status` / `INS_Flag` 全 valid 位;timestamp 来源
- §4.3 **带扰版**(noisy/biased/delayed)输出特性:每通道高斯噪声、bias、transport delay、jitter、validity 启动 transient、GPS lock 延迟、传感器健康开关
- §4.4 **参数化 knob 集合**(F1 contract = knob shape;具体 σ 数值范围由 B3 PLANT_PARAM / harness 局部 + H4 故障场景消费)
- §4.5 输出契约(消费侧字段级回引):每个 `INS_Out_Bus` 字段在两种变体下从 `Plant_States_Bus` 哪个字段衍生、如何变换
- §4.6 RNG seeding 契约(forward-cite [H3](../H-verification/H3-regression-baseline.md))
- §4.7 init / reset 行为(per [A6 §4.5.3](../A-architecture/A6-init-reset-contract.md) 消费侧 INS 协调 + §4.7.1 MIL 非对称项)
- §4.8 与 firmware mirror([B5](../B-contracts/B5-ins-bus-mirror.md))的耦合方向声明
- §4.9 与故障目录([H4](../H-verification/H4-fault-catalog.md))的 knob-shape 接口声明
- §4.10 越权显式排除清单(out-of-scope)

**不在范围(由其他工作项处理):**

- ins_stub 内部块级结构、`Plant_States_Bus → INS_Out_Bus` 字段映射的实现细节(数学算子、quaternion 转换具体子块、噪声块的拓扑) — 由 [F2 ins_stub 结构设计](F2-ins-stub-structural.md) 处理(同区,Wave 7)
- FMS / Controller 对 `INS_Out_Bus` 字段的依赖矩阵、validity 处理、fallback 行为 — 由 [F3 INS_Out_Bus 消费规则](F3-consumption-rules.md) 处理(Wave 9)
- 真实 INS 估计算法(Kalman / EKF / 互补滤波)— **firmware-owned**,架构 v1 §3 / §12.3
- `INS_Out_Bus` 字段级 schema、字段顺序、字节布局 — 由 [B1 §4.3.1](../B-contracts/B1-bus-inventory.md) 锁定 + [B5](../B-contracts/B5-ins-bus-mirror.md) mirror artifact `INS_Out_Bus.yaml` 落 firmware-canonical
- `INS_Status` / `INS_Flag` 位号 / 数值 — 由 [B2 §4.4.10 / §4.4.11](../B-contracts/B2-enum-inventory.md) 锁定
- `INS_Out_Bus` 镜像 PR 工作流、抽取脚本、版本标记 — 由 [B5](../B-contracts/B5-ins-bus-mirror.md) + [I2](../I-tooling/I2-bus-enum-mirror.md) 处理
- 故障注入场景目录、每类故障的触发条件 — 由 [H4 故障注入目录设计](../H-verification/H4-fault-catalog.md) 处理(Wave 11);F1 仅暴露 knob shape
- RNG seed 管理具体实现(种子矩阵、批量回归种子分配)— 由 [H3 回归基线策略](../H-verification/H3-regression-baseline.md) 处理(Wave 11);F1 仅声明 ins_stub 的 seed 消费形态
- 各传感器(IMU / MAG / Baro / GPS / Airspeed)的物理仿真模型 — 由 [C1 Plant 功能设计](../C-plant/C1-plant-functional.md) §"Sensor Synthesis" 处理;ins_stub **不**重新合成传感器,只直接使用 `Plant_States_Bus` truth
- 多机型 leaf 的 σ / bias 默认数值表 — 由 [B3 Parameter schema 设计](../B-contracts/B3-parameter-schema.md) PLANT_PARAM(若纳入)或 harness-local config 落地;F1 仅给"哪些 knob 必须暴露"
- harness 顶层将 ins_stub 接入 MIL 的拓扑 — 由 [G1 MIL 顶层结构](../G-harness/G1-mil-toplevel.md) 处理

## 3. 依赖

| 上游 | 引用位置 | 用途 |
|---|---|---|
| 架构 v1 §3 / §5.1.8 / §10.4 / §11(`model/harness/ins_stub/`)/ §12.3 / §15.1 / §16 第 5 项 / §17 risk 4 | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | INS firmware-owned(§3 / §12.3);`ins_stub` 是模型仓侧唯一 INS-shaped asset(§10.4 / §11 ins_stub 路径);MIL 必须通过 ins_stub 闭环(§15.1);最小执行切片含 ins_stub(§16 第 5 项);risk 4 "INS-stub realism risk" 要求"提供 noise/bias/delay 可配 knob,至少一个 MIL 场景在带扰下跑"— 本工作项是 risk 4 mitigation 的承接载体 |
| 架构 v1 §13 INS row(10 ms / 100 Hz) | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | ins_stub cadence 锁定;§4.1 |
| [A1 架构 v1 评审与缺口闭合](../A-architecture/A1-v1-review.md) (status: reviewed, 2026-05-05) | A1 §5.1.6 INS consumer dependency assigned to A3 / F3;A1 §5(F-04 widening 指派) | A1 把 INS_Out_Bus 消费方依赖矩阵分配给 A3(已 reviewed)+ F3(下游);F1 在该框架下**仅**作为 ins_stub 上游;A1 同步把 F-04 widening 的 knob 集合落地交给 F1 — 本文件 §4.3 / §4.4 即承接 |
| [A3 模块边界信号清单细化](../A-architecture/A3-module-boundaries.md) (status: reviewed, 2026-05-07) | A3 §4.3.5 `Plant_States_Bus` 字段表(ins_stub 输入);A3 §4.3.7 模拟传感器 bus(ins_stub **不**消费,只作 boundary 参照);A3 §4.4.4 `INS_Out_Bus` 消费侧字段表(ins_stub 输出 = consumer-side schema);A3 §4.8 INS 消费 dependency 矩阵 | F1 输入 / 输出字段集来源;§4.5 字段表逐行回引 A3 §4.4.4 |
| [A4 跨速率边界设计](../A-architecture/A4-rate-boundaries.md) (status: reviewed) | A4 §4.4 RB-03 / RB-04(INS 10 ms → Controller 5 ms ZOH;INS 10 ms → FMS 20 ms downsample) | ins_stub 自身周期 = 10 ms;下游消费的速率边界由 RB-03 / RB-04 处理,不在 ins_stub 内部;§4.1 |
| [A5 变体策略设计](../A-architecture/A5-variant-strategy.md) (status: reviewed) | A5 总览 — 通常机型 leaf 由 Variant Subsystem 选择,但 ins_stub 是 harness-only fixture | ins_stub 通常**不是** variant-conditional;ins_stub 自身的 ideal vs noisy/biased/delayed 选择**不**经 A5 Variant Subsystem,而是 harness-side 参数选择(§4.1.3) |
| [A6 Init/Reset 状态契约](../A-architecture/A6-init-reset-contract.md) (status: reviewed, 2026-05-05) | A6 §4.4.2 `INS_Out_Bus` 消费侧默认稳态(模型仓不约束,但消费侧必须在 `flag.<x>_valid==0` 时使用 fallback);A6 §4.5.3 INS 协调协议(消费侧 gate);A6 §4.7.1 MIL 非对称项("ins_stub 必须模拟 INS_Status / INS_Flag 位域,使 FMS 同样 gate(由 F1 实现)") | §4.7 init/reset 行为承接 A6 §4.5.3 + §4.7.1 的 explicit demand on F1;ins_stub reset → 输出稳定值且 INS_Flag transient 由本文件 §4.3.5 / §4.7 落地 |
| [A7 时间与时间戳约定](../A-architecture/A7-time-conventions.md) (status: reviewed, 2026-05-05) | A7 §4.1(单位 ms,容器 uint32_t)/ §4.2(epoch zero-from-boot)/ §4.3(wrap)/ §4.4(单调) | `INS_Out_Bus.timestamp` 字段编码;§4.2.4 / §4.5 行 `timestamp` |
| [A8 共享库块清单与归属](../A-architecture/A8-shared-library-roster.md) (status: reviewed) | A8(共享库块)— ins_stub 内部如使用 `Validity_AND` / `Stale_Detector` 等共享原语由 F2 决定;F1 不强制 | F1 只声明 knob 集合的形态,不指定使用哪些共享块 |
| [B1 Bus 清单与 schema 设计](../B-contracts/B1-bus-inventory.md) (status: reviewed, 2026-05-07) | B1 §4.3.1 `INS_Out_Bus` 消费侧 best-effort schema(字段表 / 子 schema `INS_Status` / `INS_Flag`)+ B1 §4.6.5 / §4.6.6 子 schema | F1 输出字段集 = B1 §4.3.1 字段表;在 [B5](../B-contracts/B5-ins-bus-mirror.md) mirror artifact 落地后以 mirror 为准(per B1 §4.3.1 行 260) |
| [B2 Enum 清单与数值锁定](../B-contracts/B2-enum-inventory.md) (status: reviewed) | B2 §4.4.10 `INS_Status` 值表;B2 §4.4.11 `INS_Flag` 位号表 | ins_stub 在两种变体下输出的 `INS_Status` / `INS_Flag` 值 / 位语义来源;F1 **不**重述数值,只引用 |
| [B5 INS_Out_Bus 镜像同步流程](../B-contracts/B5-ins-bus-mirror.md) (status: reviewed, 2026-05-07) | B5 §4.5 mirror artifact 存放布局(`model/shared/bus/ins/INS_Out_Bus.yaml`);B5 §4.8 F-area 下游耦合("F1 ins_stub 输出 bus 必须按本镜像产物字段顺序 / 类型 / unit / frame 输出,不得改 / 加 / 减字段") | ins_stub 输出 bus 的**精确 schema** = B5 mirror artifact;§4.5 / §4.8 |
| **co-seal Wave 6 同期 sibling**:[C1 Plant 功能设计](../C-plant/C1-plant-functional.md) *(drafting in flight)* | per [01-design-relationships.md §6 Wave 6](../01-design-relationships.md);per `C1 ⇢ F1` (§4.6) | **F1 不与 C1 形式 co-seal**(per 任务说明 "may forward-cite, NOT formal co-seal");但 ins_stub 输入 = `Plant_States_Bus`,该 bus 的字段集与设计意图来自 C1 的 Plant 输出能力。F1 字段映射对 `Plant_States_Bus` 字段的引用以 [A3 §4.3.5](../A-architecture/A3-module-boundaries.md) 字段表为权威(A3 已 reviewed),C1 仅作 forward-cite 注释;若 C1 在 reviewed 时收紧某些字段语义,F1 通过变更日志同步 |
| firmware INS 头文件 | `FMT-Firmware/src/model/ins/<variant>/lib/INS_types.h` @ `FMT-Firmware @ <pending hash>`(commit hash 在 INDEX 决策日志登记本工作项时补齐,与 [A3 §3](../A-architecture/A3-module-boundaries.md) / [A6 §3](../A-architecture/A6-init-reset-contract.md) / [A7 §3](../A-architecture/A7-time-conventions.md) / [B1 §3](../B-contracts/B1-bus-inventory.md) / [B2 §3](../B-contracts/B2-enum-inventory.md) / [B5 §3](../B-contracts/B5-ins-bus-mirror.md) 同 batch 占位约定;FMT-Firmware 仓未挂载在本环境)| `INS_Out_Bus` schema 的 firmware-canonical 真理来源,经 [B5](../B-contracts/B5-ins-bus-mirror.md) mirror artifact 进入模型仓;F1 **不**直接读 firmware 头,只通过 mirror artifact |

注:本工作项 contract_impact=**no**。ins_stub 是 harness-only fixture,**不**触及 firmware 可见 bus / enum / parameter / EXPORT 任一契约面 — `INS_Out_Bus` schema 由 [B1 §4.3.1](../B-contracts/B1-bus-inventory.md) + [B5](../B-contracts/B5-ins-bus-mirror.md) mirror artifact 锁定(那两个文档 contract_impact=yes),F1 只是该 schema 的**消费 + 生产**两侧实现的功能侧设计;ins_stub 自身不会被导出到 firmware(架构 v1 §5.1.8 / §10.4 / §17 risk 4 mitigation paragraph 显式声明 stub never exported)。

## 4. 设计内容

### 4.1 ins_stub 的目的、boundary 与变体集合

#### 4.1.1 目的

ins_stub 把 Plant 输出的 truth 状态(`Plant_States_Bus`)转化成一个**字段级 schema-compliant 的 `INS_Out_Bus`**,以替代 firmware 真实 INS 在 MIL / SIL 仿真路径下的位置 — 让 FMS / Controller 在不依赖 firmware C/C++ 估计器的前提下闭合控制环。架构 v1 §15.1 把 ins_stub 列为 MIL "minimum scope" 的必需件;架构 v1 §16 第 5 项把它列为最小执行切片必需件;架构 v1 §17 risk 4 把 "ins_stub realism risk"(过度理想化掩盖真实问题)列为顶级风险,要求暴露 noise / bias / delay knob。

#### 4.1.2 Boundary(严格 harness-only)

| 维度 | 约束 |
|---|---|
| **位置** | `model/harness/ins_stub/`(架构 v1 §6 / §11 推荐目录树明确给路径;架构 v1 §10.4 重申 ins_stub 是 verification fixture 而非 architecture layer 的一部分)|
| **执行环境** | **MIL / SIL only**;真实 firmware 飞行运行时使用 firmware 内 hand-written INS C/C++(架构 v1 §3 / §12.3) |
| **导出** | **永不**导出到 `export/firmware/`(架构 v1 §6 树:`export/firmware/` 下无 `ins/` 目录;架构 v1 §10.4 "the only INS-shaped asset is the harness `ins_stub`, which is a verification fixture and not part of [shared / module-shared / vehicle leaf] layer";架构 v1 §17 risk 4 mitigation:"the stub is **never** exported to firmware") |
| **契约面参与** | **不参与**任何 firmware 契约 diff(B4 / I3 不对 ins_stub 进行 diff);ins_stub 的"契约"只是"输出 schema 必须等于 [B5](../B-contracts/B5-ins-bus-mirror.md) mirror artifact" |
| **被消费方** | 仅 MIL harness(由 [G1](../G-harness/G1-mil-toplevel.md) 顶层接入);消费下游 = FMS / Controller(per [A3 §4.4.4 + §4.5.3](../A-architecture/A3-module-boundaries.md))— 但 FMS / Controller 看不到 ins_stub 本身,只看到 `INS_Out_Bus` |
| **谁拥有 RNG 状态** | ins_stub **不**自有 RNG;noise / bias / jitter 由外部 seed 经 PLANT_PARAM 或 harness override 注入(§4.6) |
| **是否参与 SIL** | ins_stub **不**参与 SIL(架构 v1 §15.2 SIL 范围 = Plant / FMS / Controller only;ins_stub 是 verification fixture)|

#### 4.1.3 变体集合(per 审计 [F-04 / F-28](../_audit/2026-05-05-plan-audit.md))

ins_stub 暴露**两个**变体,通过 harness-side 参数选择:

| 变体 ID | 名称 | 使用场景 |
|---|---|---|
| **F1-V-IDEAL** | 理想版(ideal) | golden-path 场景、回归基线、CI sanity check;让 FMS / Controller 在"无 INS 噪声"前提下展示算法基线行为 |
| **F1-V-NOISY** | 带扰版(noisy / biased / delayed) | 噪声拒绝 sanity check、F3 fallback 测试、H4 故障目录场景、架构 v1 §17 risk 4 要求的"至少一个 MIL 场景在带扰下跑"(promote to SIH 前置闸口) |

变体选择**不**经 [A5](../A-architecture/A5-variant-strategy.md) Variant Subsystem 机制(A5 主要服务于多机型 leaf 与 phase 路线;ins_stub 是 harness fixture,不属于机型 leaf 范畴)。变体在 harness 层通过参数(布尔 flag 或 enum,由 [G1](../G-harness/G1-mil-toplevel.md) 选定具体载体)二选一;[F2](F2-ins-stub-structural.md) 决定结构上是用 enabled subsystem / variant subsystem / 还是 if-action 块来实现该选择。F1 仅承诺"两种变体在同一 ins_stub 模型内可被切换,且切换不需修改 Plant / FMS / Controller"。

#### 4.1.4 Cadence(10 ms / 100 Hz)

ins_stub 输出 `INS_Out_Bus` 的 cadence = **10 ms / 100 Hz**,与架构 v1 §13 INS row("INS: 10 ms / 100 Hz, firmware-owned wrapper / system integration cadence")严格一致 — 让模型仓侧消费者(FMS / Controller)看到的 sample rate 与 firmware 实际 INS 完全相同(避免 SIH 阶段速率 mismatch)。下游跨速率边界由 [A4 §4.4 RB-03 / RB-04](../A-architecture/A4-rate-boundaries.md) 处理,**不在** ins_stub 自身范围内。

ins_stub 输入侧:`Plant_States_Bus` cadence = 1 ms(per [A3 §4.3.5](../A-architecture/A3-module-boundaries.md) Plant 输出 1 ms);ins_stub 内部需把 1 ms 输入降速为 10 ms 输出。具体降速机制(sample-time-based downsample 还是 explicit decimation)由 [F2](F2-ins-stub-structural.md) 决定;F1 仅承诺"输出 cadence = 10 ms"。

### 4.2 理想版(F1-V-IDEAL)输出特性

#### 4.2.1 通用约束

理想版的输出语义是:**identity transform from `Plant_States_Bus` truth → `INS_Out_Bus`**,叠加(a)单位 / 坐标系转换(quaternion 已 NED→body,无需变换;欧拉角由 quaternion 派生;LLA 经纬度由 NED 位置 + home 派生 — 见 §4.5);(b)字节级类型对齐(`Plant_States_Bus` 多用 `single`,`INS_Out_Bus.position_lla` 用 `double`,需类型 cast)。**不**注入任何噪声、bias、delay。

#### 4.2.2 Validity 行为

| 项 | F1-V-IDEAL 行为 |
|---|---|
| `INS_Status` | 自 reset 完成第一个 step 起即输出 `INS_STATUS_READY`(per [B2 §4.4.10](../B-contracts/B2-enum-inventory.md) 数值)— 即 ready 位**立即**置位;**不**经过 `INS_STATUS_NOT_READY` / `INS_STATUS_INITIALIZING` 的过渡(对比带扰版 §4.3.5)|
| `INS_Flag.position_valid` | 立即 1 |
| `INS_Flag.velocity_valid` | 立即 1 |
| `INS_Flag.attitude_valid` | 立即 1 |
| `INS_Flag.heading_valid` | 立即 1 |
| `INS_Flag.mag_valid` | 立即 1 |
| `INS_Flag.gps_valid` | 立即 1(理想版无 GPS lock latency)|
| `INS_Flag.baro_valid` | 立即 1 |
| `INS_Flag.airspeed_valid` | 1(若 fixed-wing variant)/ 0(若 multicopter variant,airspeed 通道未启用)— 由 [F2](F2-ins-stub-structural.md) 在结构上决定如何 conditional;F1 锁定 multicopter Phase 2 默认 0 |
| `INS_Flag.vel_body_valid` | 立即 1(若 firmware mirror 含此位 per [B2 §4.4.11](../B-contracts/B2-enum-inventory.md);否则 N/A)|

理由:理想版的目的是让 FMS / Controller 在"完美估计"前提下跑;任何 validity transient 都属于带扰版的语义。

#### 4.2.3 Timestamp

`INS_Out_Bus.timestamp` 字段:per [A7 §4.1 / §4.2 / §4.3 / §4.4](../A-architecture/A7-time-conventions.md):

- 单位:`ms`(firmware-legacy 无后缀)
- 容器:`uint32_t`
- Epoch:仿真启动 t=0(zero-from-boot)
- 单调:严格非递减,与 ins_stub 自身的 sample hit 单调对应
- Wrap:模 \(2^{32}\) 自然 wrap(无特殊处置;49.71 天上限对单次 MIL 仿真足够)

理想版的 timestamp **直接来源于 ins_stub 自身的 step 触发时刻**(harness 调度的 sample boundary);**不**从 `Plant_States_Bus.timestamp` 派生(那是 1 ms cadence,与 INS 10 ms cadence 错位)。具体 timestamp 生成实现细节由 [F2](F2-ins-stub-structural.md) 决定。

#### 4.2.4 适用场景

- golden-path closed-loop:arm → takeoff → hover → manual / position hold → land → disarm(架构 v1 §16 第 6 项最小切片)
- 回归基线:理想版的输出必须 byte-equal 复现给定相同 `Plant_States_Bus` 序列(per A6 §"deterministic" 定义),作为 H3 baseline 的 stub 侧锚点
- CI sanity check:每次模型仓 PR 在 ins_stub ideal 下跑一个 minimal scenario,验证闭环未坏

### 4.3 带扰版(F1-V-NOISY)输出特性

#### 4.3.1 通用约束

带扰版在理想版基础上叠加噪声 / bias / 延迟 / jitter / validity transient / 传感器健康开关 — 每一项都是**可参数化**的(§4.4 knob 集合),允许从"接近理想"(σ=0,bias=0,delay=0)平滑滑到"严苛带扰"(σ 大、bias 漂移、delay 多 sample)。

#### 4.3.2 噪声(每通道高斯)

对 `INS_Out_Bus` 中的 navigation 字段(position / velocity / quaternion / euler / ang_rate / acc),每通道叠加**零均值高斯噪声**:

```text
output_field[i] = ideal_field[i] + N(0, σ_field[i])
```

- σ 值由 §4.4 knob 暴露(F1 contract = knob shape;具体 σ 数值范围由 H4 fault catalog / B3 PLANT_PARAM 或 harness-local 配置)
- noise 时间序列**确定性**:同一 seed + 同一场景 → 同一 noise 序列(per §4.6 RNG 契约)
- noise 在每个 INS 10 ms sample 独立采样(white noise);若需 colored noise(low-pass 后的 filtered noise),由 [F2](F2-ins-stub-structural.md) 在结构上决定是否在共享库 `Filters/`([A8](../A-architecture/A8-shared-library-roster.md))中加 filter;F1 仅承诺 white-noise knob 必须存在

#### 4.3.3 偏差(constant + random-walk)

每通道 bias 由**两部分**构成:

```text
bias[t] = bias_const + bias_walk[t]
bias_walk[t] = bias_walk[t-1] + N(0, σ_walk × √dt_s)
output_field[i] = ideal_field[i] + bias[i, t]
```

- `bias_const`:每通道一个常量(场景配置时锁定)
- `bias_walk`:随机游走漂移(模拟陀螺 / 加速度计 bias drift)
- σ_walk → 0 即退化为纯常量 bias;σ_walk 大 → 可观察长期漂移
- 两部分均由 §4.4 knob 暴露
- bias_walk 的 random-walk 状态在 ins_stub 内部 **per-channel** 持有;reset 时按 §4.7 / 用户场景设置(可选 zero、可选 preserve)— F1 默认 reset 后 `bias_walk = 0`(回到只剩 `bias_const`)

#### 4.3.4 Transport delay(per channel)

每通道支持**可配 transport delay**:

- 单位:samples(以 INS 10 ms cadence 计)或 ms(harness 表达上等价)
- 范围:0..N samples(N 上限由 [F2](F2-ins-stub-structural.md) 在结构上选 ring-buffer 大小决定;F1 承诺至少支持 0..10 samples = 0..100 ms)
- delay knob 默认 = 0(理想版退化语义);带扰场景 typical 1..3 samples
- 不同字段允许不同 delay(F3 fallback 测试中可能需要"position 比 attitude 滞后"的场景)— knob 是 **per-field**

实现注意:transport delay 是 ZOH(zero-order hold)on a buffer,**不是** rate-based 模拟;F1 锁定 ZOH 语义,F2 选择具体块(`Delay` 块 vs `Tapped Delay` 块由 F2 定)。

#### 4.3.5 Validity 启动 transient(per [A6 §4.5.3 / §4.7.1](../A-architecture/A6-init-reset-contract.md))

带扰版**必须**模拟真实 INS 的"启动后非立即就绪"行为,以让 FMS 的 INS readiness gate(per A6 §4.5.3 第 2 项)在 MIL 路径下有相同语义路径(对称 firmware 路径 — 这是 [A6 §4.7.1](../A-architecture/A6-init-reset-contract.md) 行 "ins_stub 必须模拟 INS_Status / INS_Flag 位域,使 FMS 同样 gate(由 F1 实现)" 的 explicit demand on F1)。

reset 后 timeline:

```text
t = 0          : INS_Status = INS_STATUS_NOT_READY
                 INS_Flag = 0 (all bits 0)
t = T_init     : INS_Status → INS_STATUS_INITIALIZING
                 attitude_valid → 1 (惯性传感器先稳定)
                 heading_valid → 1
                 velocity_valid → 1 (body-frame from IMU integration)
t = T_pos_lock : position_valid → 1 (NED 位置锁定;依赖 GPS / baro 融合 in firmware)
                 baro_valid → 1
                 mag_valid → 1
t = T_gps_lock : gps_valid → 1
                 INS_Status → INS_STATUS_READY
```

- 三段 timing(T_init / T_pos_lock / T_gps_lock)由 §4.4 knob 暴露
- 默认值(F1 给出 shape;具体 ms 数由 harness 场景或 B3 PLANT_PARAM 落):T_init typical 200..500 ms;T_pos_lock typical 1..3 s;T_gps_lock typical 5..30 s(模拟冷启动 GPS)
- 理想版(§4.2.2)不进入此 transient — 所有 valid 位 t=0 即 1

#### 4.3.6 Measurement-rate variation(jitter)

可选 knob:

- `jitter_ms`:在 nominal 10 ms sample 上叠加 ±N ms 抖动(模拟 firmware 调度抖动)
- 默认 0(无 jitter);典型 0..2 ms
- jitter 仅影响 `timestamp` 字段(让消费侧 stale detector 看到非完美 10 ms 周期);**不**改变 ins_stub 的 Simulink sample hit(那由 harness 调度强制 10 ms;jitter 只是 timestamp 字段的人工扰动)
- F2 决定 jitter 的 RNG 来源(同 §4.6 seed 体系)

#### 4.3.7 GPS lock latency

独立于 §4.3.5 整体 transient 的细化项:

- `T_gps_lock_ms`:从 reset 起 N ms 内 `INS_Flag.gps_valid = 0`,且 `INS_Out_Bus.position_lla`(LLA)字段输出 zero / NaN(具体由 F2 决定输出何值,F1 锁"未 lock 期间 LLA 不可用")
- N 默认 typical 5000..30000 ms;modeling cold-start 与 hot-start 差异由场景脚本设置不同 N

#### 4.3.8 Sensor health 开关(布尔)

每个传感器源(MAG / BARO / GPS / AIRSPEED)暴露独立**健康布尔开关**:

| Knob | 行为(false 时) |
|---|---|
| `mag_healthy` | `INS_Flag.mag_valid` 强制 0;`INS_Status` 可能降到 `INS_STATUS_DEGRADED`(per [B2 §4.4.10](../B-contracts/B2-enum-inventory.md))|
| `baro_healthy` | `INS_Flag.baro_valid` 强制 0;`INS_Out_Bus.position_lla[2]`(alt)若 INS 在 firmware 用 baro fuse,本模型选择继续输出但不可信;F1 锁 flag 行为,fallback 由 F3 |
| `gps_healthy` | `INS_Flag.gps_valid` 强制 0;`INS_Out_Bus.position_lla` LLA 输出 zero / NaN(per §4.3.7);position_valid 在 GPS 失效后是否 hold 由 F2 决定(典型行为:hold 短期再失效) |
| `airspeed_healthy` | `INS_Flag.airspeed_valid` 强制 0(仅 fixed-wing variant 适用) |

这些开关是**bool**,可在场景中点动态切换以模拟传感器中途失效 — 这是 H4 故障目录的核心 knob。

#### 4.3.9 适用场景

- 噪声-拒绝 sanity:验证 controller 对小 σ 噪声的抑制能力
- F3 fallback 测试:让 `INS_Flag.position_valid` 中途归零,验证 FMS / Controller 按 [F3](F3-consumption-rules.md) 规则降级(F3 自身设计 forward-cite)
- H4 故障目录:每条 H4 故障场景设置一组 knob 值(具体场景设计 forward-cite [H4](../H-verification/H4-fault-catalog.md))
- promote-to-SIH 闸口:架构 v1 §17 risk 4 mitigation "至少一个 MIL 场景在带扰下跑",带扰版是该闸口的承接

### 4.4 参数化 knob 集合(F1 contract = knob shape)

本节锁定 **knob shape**(每个 knob 的语义、类型、默认行为类别);**不**锁定具体 σ / delay 默认数值范围 — 数值由 [B3 PLANT_PARAM](../B-contracts/B3-parameter-schema.md)(若纳入)或 harness-local 场景配置 + [H4 故障目录](../H-verification/H4-fault-catalog.md) 场景表 + [H1 验证场景目录](../H-verification/H1-scenario-catalog.md) 三方消费决定。F1 的承诺:**ins_stub 必须暴露这些 knob,让 H4 / H1 / B3 据此填数值**。

#### 4.4.1 Knob 总表(shape contract)

| Knob ID | 类型 | 单位 | 应用通道 | 默认行为类别(per [A6 §4.4.1](../A-architecture/A6-init-reset-contract.md)) | 备注 |
|---|---|---|---|---|---|
| **σ_noise** | float | per-field 单位 | per-field(pos_ned / vel_ned / quat / euler / ang_rate / acc / position_lla) | PARAM(默认 0 = ideal) | 高斯白噪声标准差,per-channel 独立;§4.3.2 |
| **bias_const** | float | per-field 单位 | per-field(同上,加 mag / baro / gps 派生通道) | PARAM(默认 0) | 常量 bias;§4.3.3 |
| **σ_walk** | float | per-field 单位 / √s | per-field(同 bias_const) | PARAM(默认 0) | random-walk 漂移率;§4.3.3 |
| **delay_samples** | int | samples (10 ms 单位) | per-field(7 个 navigation 字段)| PARAM(默认 0) | transport delay;§4.3.4;F1 承诺范围 0..10 samples |
| **T_init_ms** | int | ms | scalar(全 INS) | PARAM(默认 0 in ideal;典型 200..500 in noisy) | reset 后 → INITIALIZING 的 latency;§4.3.5 |
| **T_pos_lock_ms** | int | ms | scalar | PARAM(默认 0;典型 1000..3000) | reset 后 → position_valid=1 的 latency;§4.3.5 |
| **T_gps_lock_ms** | int | ms | scalar | PARAM(默认 0;典型 5000..30000) | reset 后 → gps_valid=1 的 latency;§4.3.7 |
| **jitter_ms** | float | ms | scalar(applies to timestamp) | PARAM(默认 0) | timestamp 抖动幅度;§4.3.6 |
| **mag_healthy** | bool | N/A | scalar | HARDCODED true(健康) | §4.3.8 |
| **baro_healthy** | bool | N/A | scalar | HARDCODED true | §4.3.8 |
| **gps_healthy** | bool | N/A | scalar | HARDCODED true | §4.3.8 |
| **airspeed_healthy** | bool | N/A | scalar(fixed-wing variant only) | HARDCODED true / N/A | §4.3.8;multicopter Phase 2 N/A |
| **rng_seed** | uint32 | N/A | scalar(seed for all noise / walk / jitter) | PARAM(无默认;场景必须显式提供) | §4.6 RNG 契约 |
| **variant** | enum {ideal, noisy} | N/A | scalar(选择 §4.1.3 两变体) | HARDCODED ideal | §4.1.3 |

#### 4.4.2 Knob 数值范围(shape vs 数值的边界)

F1 **不**锁定具体数值范围(例如 "σ_noise[pos_ned] ∈ [0.05, 0.5] m"):

- 多旋翼具体 σ 来自 multicopter 工程经验 / firmware 估计器实际表现 — 落在 [B3 PLANT_PARAM](../B-contracts/B3-parameter-schema.md)(若 B3 决定纳入 ins_stub 配置项)或 harness-local YAML 场景表
- 故障场景的 knob 值矩阵(如 "GPS 失效 30 秒、MAG 偏差跳变 0.5 Gauss")由 [H4](../H-verification/H4-fault-catalog.md) 场景表锁
- H1 golden-path 场景与 noisy stress 场景的 knob 配置由 [H1](../H-verification/H1-scenario-catalog.md) 锁

F1 仅承诺:每个 knob 是**runtime-tunable**(per [A6 §4.4.1](../A-architecture/A6-init-reset-contract.md) PARAM 类别),可在场景启动时由 harness 注入;凡 knob 默认值导致退化为理想版的(σ=0、bias=0、delay=0、health=true、T_*_ms=0)的组合,行为必须 byte-equal 于 §4.2 理想版。

#### 4.4.3 Knob 命名约定(传递给 [A2](../A-architecture/A2-naming-conventions.md))

knob 在 PLANT_PARAM 或 harness 配置中的字段名建议约定(F1 仅给建议,A2 / B3 决定是否采纳):

- per-field knob 用 `ins_stub.<knob_id>.<field_name>` 路径风格(例如 `ins_stub.sigma_noise.pos_ned_m`)
- scalar knob 用 `ins_stub.<knob_id>` 路径风格(例如 `ins_stub.t_gps_lock_ms`)
- bool knob 用 `ins_stub.<knob_id>` 路径风格(例如 `ins_stub.mag_healthy`)

A2 命名规范若未来对此区域加约束,F1 通过变更日志同步。

### 4.5 输出契约(消费侧字段级)

本节给 `INS_Out_Bus` **每个被消费字段**的:(a)在 `Plant_States_Bus` 中的 truth 来源字段;(b)理想版的变换;(c)带扰版叠加的 knob。**字段名 / 类型 / 顺序**以 [B5](../B-contracts/B5-ins-bus-mirror.md) mirror artifact `INS_Out_Bus.yaml` 为权威;本节字段集与 [B1 §4.3.1](../B-contracts/B1-bus-inventory.md) + [A3 §4.4.4](../A-architecture/A3-module-boundaries.md) 一致(消费侧 best-effort schema)。

| `INS_Out_Bus` 字段 | `Plant_States_Bus` truth 来源 | 理想版变换(F1-V-IDEAL) | 带扰版叠加(F1-V-NOISY) | 备注 |
|---|---|---|---|---|
| `position_lla[3]` (lat_deg, lon_deg, alt_m;double) | `pos_ned_m[3]`(NED single)+ home_lat/lon/alt(harness 配置) | NED → LLA 转换(WGS-84 平面近似 or full ellipsoid;F2 选);type cast single → double | + σ_noise[position_lla] + bias_const[position_lla] + bias_walk[position_lla] + delay_samples[position_lla];gps_healthy=false 时 LLA → zero/NaN per §4.3.7 / §4.3.8 | `position_lla` 在 firmware 端由 GPS lock 后才有效;ideal 即时给,noisy 模 GPS lock latency |
| `position_ned_m[3]` (single) | `pos_ned_m[3]` | identity(同类型同坐标系) | + σ_noise[pos_ned] + bias_const[pos_ned] + bias_walk[pos_ned] + delay_samples[pos_ned] | 主位置消费字段(per [A3 §4.4.4](../A-architecture/A3-module-boundaries.md))|
| `velocity_ned_mps[3]` (single) | `vel_ned_mps[3]` | identity | + σ_noise[vel_ned] + bias[vel_ned] + delay[vel_ned] | |
| `quat_ned_to_b[4]` (single) | `quat_ned_to_b[4]` | identity(quaternion 已 NED→body)| + σ_noise[quat](注:加噪后需 normalize 维持 unit-length;F2 实现)+ delay[quat] | bias 在 quaternion 上不直接有意义;F1 不暴露 bias_const[quat],但暴露 σ_noise + delay |
| `euler_ned_to_b_rad[3]` (single, yaw/pitch/roll) | `euler_ned_to_b_rad[3]`(若 Plant 输出)/ 由 quat 派生 | identity 或 quat → euler conversion(F2 选) | + σ_noise[euler] + bias_const[euler] + bias_walk[euler] + delay[euler] | firmware-legacy 字段名待 [B5](../B-contracts/B5-ins-bus-mirror.md) verify |
| `ang_rate_b_radps[3]` (single, body) | `ang_rate_b_radps[3]` | identity(body-frame 已对齐)| + σ_noise[ang_rate] + bias_const[ang_rate] + bias_walk[ang_rate] + delay[ang_rate] | 角速度 bias 是真实陀螺常见误差源 |
| `acc_b_mps2[3]` (single, body) | `acc_b_mps2[3]`(body-frame specific force)| identity | + σ_noise[acc] + bias_const[acc] + bias_walk[acc] + delay[acc] | 加速度 bias 是真实加速度计常见误差 |
| `INS_Status` (uint32 bitfield / enum) | N/A(stub-internal state machine,per §4.2.2 / §4.3.5)| `INS_STATUS_READY` 立即(per [B2 §4.4.10](../B-contracts/B2-enum-inventory.md))| 启动 transient: NOT_READY → INITIALIZING → READY(per §4.3.5);失健 → DEGRADED / FAULT 视场景(per §4.3.8)| 数值由 B2 锁;F1 不重述 |
| `INS_Flag` (uint32 bitfield) | N/A(stub-internal state)| 全位立即 1(per §4.2.2)| 各位按 §4.3.5 timeline + §4.3.8 health 开关动态置位 / 清零 | 位号由 [B2 §4.4.11](../B-contracts/B2-enum-inventory.md) 锁;F1 不重述 |
| `timestamp` (uint32) | N/A(由 ins_stub 自身 step 触发时刻派生,per §4.2.3)| ms-uint32,zero-from-boot,严格单调(per [A7](../A-architecture/A7-time-conventions.md))| + jitter_ms(§4.3.6,可正可负)| ins_stub 不**直接消费** `Plant_States_Bus.timestamp`(那是 1 ms cadence,与 INS 10 ms 错位)|

#### 4.5.1 派生通道(GPS / MAG / BARO / Airspeed 在 INS_Out_Bus 内的呈现)

注意:**ins_stub 不消费 Plant 的 `IMU_Bus` / `MAG_Bus` / `Barometer_Bus` / `GPS_uBlox_Bus` / `AirSpeed_Bus`**(那些是 [C1 Plant 功能设计](../C-plant/C1-plant-functional.md) 的 sensor synthesis 输出,目标是给 firmware INS 驱动消费;ins_stub 走捷径直接用 `Plant_States_Bus` truth)。`INS_Out_Bus` 中:

- `position_lla` 是 ins_stub 由 `Plant_States_Bus.pos_ned_m` + harness home 直接派生,**不**经 GPS bus
- mag / baro / airspeed 在 firmware INS_Out_Bus 中**通常不直接呈现为 navigation 字段**(它们是估计器内部融合的输入,经融合后体现在 attitude / position 中);若 mirror artifact 显示 firmware INS_Out_Bus 含独立的 mag / baro / airspeed 字段,F1 通过变更日志补行;否则上表覆盖完整字段集

mag / baro / gps / airspeed 的"健康"概念在 F1 中只通过 `INS_Flag.mag_valid` / `baro_valid` / `gps_valid` / `airspeed_valid` 位与 `INS_Status` 的 DEGRADED 等级体现(§4.3.8),**不**对应独立的 navigation 字段。

#### 4.5.2 Self-consistency 约束(per [A6 §4.4.3](../A-architecture/A6-init-reset-contract.md))

A6 §4.4.3 要求 "Plant 输出与 ins_stub 输出在 init 后必须 self-consistent"(若 Plant 初始 hover,ins_stub 在 init 后应反映 hover 位置)— **F1 由 §4.5 字段映射表的 identity 关系自然满足**。F2 实现必须保证理想版退化下 init 后第一个 sample 的 `INS_Out_Bus.position_ned_m` 等于 `Plant_States_Bus.pos_ned_m` 的 truth 值(byte-equal 在 single 精度下)。

### 4.6 RNG seeding 契约(forward-cite [H3](../H-verification/H3-regression-baseline.md))

per 审计 [F-05](../_audit/2026-05-05-plan-audit.md)("RNG seeding / simulation determinism not specified")+ [A6 §4.2.2 行 105](../A-architecture/A6-init-reset-contract.md)("若 ins_stub 需 RNG,由 H3 RNG 契约 + I 区脚本管理,不在 init 内"):

#### 4.6.1 ins_stub 不持有 RNG state

- ins_stub **不**在自身 init 中调用 RNG(违反 [A6 §4.2.2](../A-architecture/A6-init-reset-contract.md) "禁止生成随机数 / 调用 RNG"约束 — 注:此约束针对 `*_init` 函数;ins_stub 不是 firmware-export 模块,但 F1 把同样的"RNG 不在 init"原则套用,以便 reset 复现)
- ins_stub 在 step 内消费 noise 序列(§4.3.2 / §4.3.3 / §4.3.6)— 该序列**必须**从外部 seed deterministic 派生

#### 4.6.2 Seed 来源

- `rng_seed`(uint32)是 §4.4.1 表中暴露的 knob;由 harness 在场景启动时注入(具体注入路径:PLANT_PARAM 字段 / harness YAML / Simulink mask param,由 [F2](F2-ins-stub-structural.md) + [G1](../G-harness/G1-mil-toplevel.md) + [G4](../G-harness/G4-pilot-injection.md) 决定)
- F1 承诺:**给定相同 `rng_seed` + 相同场景脚本 + 相同 PLANT_PARAM**,ins_stub 在带扰版下产生**byte-equal** `INS_Out_Bus` 序列(deterministic 复现,per [A6 §4.1 deterministic](../A-architecture/A6-init-reset-contract.md))

#### 4.6.3 多 noise 通道的 seed 派生

ins_stub 内部多个 noise / walk / jitter 通道(per §4.3),每通道需独立 noise 流以避免 cross-channel 相关。F1 承诺:

- 每通道 noise 流由 `rng_seed` 通过 deterministic sub-seed 派生(具体派生函数:`sub_seed_<channel> = hash(rng_seed, channel_id)` 或等价 split;由 [F2](F2-ins-stub-structural.md) 实现)
- 跨场景 / 跨变体的 sub-seed 派生规则统一,避免 H3 baseline 不可比

#### 4.6.4 与 [H3 §"RNG seed 管理契约"](../H-verification/H3-regression-baseline.md) 的承接(forward-cite,Wave 11)

H3(Wave 11)将定义:

- 种子来源(场景脚本字段 / 全局 PARAM / CLI 参数)
- 固化方式(seed 矩阵存档于 baseline `.mat`)
- 批量回归时的种子矩阵(每场景多 seed → covariance estimate)

F1 承接义务:**ins_stub 必须接受 H3 落定的 seed 注入接口**;F1 当前只暴露 `rng_seed`(uint32 scalar)作为最小契约;若 H3 决定使用更复杂的 seed 集合(例如 per-channel 独立 seed array),F1 通过变更日志同步扩展 knob 集合。

### 4.7 Init / reset 行为(per [A6 §4.5.3 / §4.7.1](../A-architecture/A6-init-reset-contract.md))

#### 4.7.1 ins_stub init 行为

- ins_stub 是 harness-only fixture,**不**有 firmware-export `*_init` 函数;但有 Simulink 内部 init 行为(初值 / unit-delay / state)
- init 后第一次 step 之前,ins_stub 内部 state(noise RNG sub-seed initial state、bias_walk 状态、delay buffer 状态、validity transient timer)必须 deterministic
- init 后第一次 sample 输出**取决于变体**:
  - F1-V-IDEAL:输出立即 valid 的 `INS_Out_Bus`(per §4.2)
  - F1-V-NOISY:输出 `INS_Status = INS_STATUS_NOT_READY`,`INS_Flag` 全 0,navigation 字段值未定义(消费侧 gate 守住,per [F3](F3-consumption-rules.md))

#### 4.7.2 ins_stub reset 行为

按 [A6 §4.4.2 行"INS_Out_Bus 消费侧默认"](../A-architecture/A6-init-reset-contract.md)("status / flag … INS firmware-owned;模型仓不约束其值,但**必须**消费 `ready` / `*_valid` 位作为 gate"),ins_stub reset 行为:

| 触发源(per [A6 §4.3.1](../A-architecture/A6-init-reset-contract.md)) | ins_stub 行为 |
|---|---|
| **TS-MIL-HARNESS**(MIL only;harness 显式触发) | 完整 reset:noise RNG 重新从 `rng_seed` 派生(determinism 关键);bias_walk → 0(回到只剩 bias_const);delay buffer 清空;validity transient timer 归零 → 重新 NOT_READY → INITIALIZING → READY 序列(noisy variant)或立即 READY(ideal variant) |
| **TS-FMS-OUT / TS-FAILSAFE / TS-EXT-CMD / TS-MODE-XCHG**(per A6 §4.3.1) | ins_stub **不响应**这些信号 — 它模拟 firmware-owned INS,而 firmware INS 不被 FMS reset 联动(per [A6 §4.5.2 第 2 项 INS 协调](../A-architecture/A6-init-reset-contract.md))。ins_stub 持续输出当前估计 |

#### 4.7.3 与 A6 §4.5.3 INS 协调协议的对称

[A6 §4.5.3](../A-architecture/A6-init-reset-contract.md) 定义 firmware 路径下 INS 协调:

> FMS 在每个 step 内,首先检查 `INS_Flag.bit.ready`;若为 0,保持 disarmed/idle 状态。

F1 承接:ins_stub 必须**让 FMS 在 MIL 路径下走相同代码路径** — 即 ins_stub 的 `INS_Status` / `INS_Flag` 位**真实可能为 0**(noisy variant 的启动 transient 期间;health 开关失效时;延迟期间)— 让 FMS 的 readiness gate 在 MIL 中能被实际触发并被验证。理想变体不能用于"INS readiness gate 测试"场景,这是 A6 §4.7.1 行 "ins_stub 必须模拟 INS_Status / INS_Flag 位域,使 FMS 同样 gate(由 F1 实现)" 的核心 demand。

#### 4.7.4 与 A6 §4.7.1 MIL 非对称项的对称

A6 §4.7.1 表行 "INS 是否被建模" 的处置 = "消费侧契约一致(§4.5.3);ins_stub 必须模拟 INS_Status / INS_Flag 位域,使 FMS 同样 gate(由 F1 实现)" — F1 §4.3.5(transient timeline)+ §4.3.8(health switches)+ §4.7.2(reset 行为)是该 demand 的承接载体。

### 4.8 与 firmware mirror([B5](../B-contracts/B5-ins-bus-mirror.md))的耦合方向

#### 4.8.1 ins_stub 消费 mirror artifact

ins_stub 输出 bus 的 schema(字段名 / 类型 / 字节宽度 / 累计 offset / 单位 / 坐标系 / encoding)= [B5 §4.5 mirror artifact](../B-contracts/B5-ins-bus-mirror.md) `model/shared/bus/ins/INS_Out_Bus.yaml` — **完全相等**,**不许**新增 / 删除 / 重排 / 改类型字段(per [B5 §4.8](../B-contracts/B5-ins-bus-mirror.md) 行 "ins_stub 必须按本镜像产物字段顺序 / 类型 / unit / frame 输出,不得改 / 加 / 减字段")。

#### 4.8.2 firmware schema 升级时 ins_stub 跟随

当 firmware bump `INS_Out_Bus` schema(per [B5 §4.4 ins_bus_schema_version](../B-contracts/B5-ins-bus-mirror.md) bump 规则)→ B5 流程产出新 mirror PR → 模型仓 mirror artifact 更新 → ins_stub 必须在**下一个 harness rebuild** 时按新 schema 重新生成 bus 对象 + 字段映射。具体 rebuild 流程由 [I1 init 脚本](../I-tooling/I1-init-script.md) + [G1](../G-harness/G1-mil-toplevel.md) 落地。**F1 不设计 rebuild 流程**;F1 只声明依赖方向:`ins_stub → mirror artifact`(消费方向)。

#### 4.8.3 字段集变化对 §4.4 knob 集合的影响

若 mirror artifact 在 firmware schema 升级后新增 navigation 字段(例如 firmware 后续在 INS_Out_Bus 加 `wind_estimate_b_mps[3]`),则 §4.4.1 knob 表必须扩展(添加该字段的 σ_noise / bias / delay knob)。该扩展由 F1 后续修订处理 — F1 当前 §4.4.1 表覆盖 [A3 §4.4.4](../A-architecture/A3-module-boundaries.md) 已枚举的 7 个 navigation 字段,与 firmware 当前 best-effort 推断一致。

### 4.9 与 [H4 故障注入目录](../H-verification/H4-fault-catalog.md) 的耦合(下游,Wave 11)

per [01-design-relationships.md §4.10 audit-added](../01-design-relationships.md) 行 "H4 ⇢ F1"(故障注入与 ins_stub 噪声/偏差/延迟知识共享)+ 审计 [F-04](../_audit/2026-05-05-plan-audit.md)("Widen F1 noise/bias/delay knobs **or** add H4 fault catalog" — 该审计已选择**两条都做**:F1 加 knob,H4 加场景目录):

#### 4.9.1 F1 提供的 knob shape = H4 fault catalog 的最低层

H4(Wave 11)将设计故障场景目录,每条场景对应一组 ins_stub knob 值。典型映射:

| H4 故障类(forward-cite) | F1 knob 设置 |
|---|---|
| **传感器失效:GPS 失效 30 秒** | `gps_healthy = false` for 30 s window;期间 `gps_valid = 0`,`position_lla` 不可用 |
| **传感器失效:MAG 失效** | `mag_healthy = false`;`mag_valid = 0`,`INS_Status` 可降级 DEGRADED |
| **传感器失效:BARO 失效** | `baro_healthy = false`;`baro_valid = 0` |
| **估计器降级:bias 漂移** | `σ_walk[ang_rate] = X` 或 `bias_const[ang_rate]` step change mid-scenario |
| **估计器降级:noise 突增** | `σ_noise[*]` step change mid-scenario(模拟 sensor 振动 / 干扰)|
| **链路降级:INS 时间戳抖动** | `jitter_ms` 增大;timestamp staleness gate 测试 |
| **Validity 丢失** | `INS_Flag.<bit> = 0` 强制(通过 health 开关或独立 force-flag knob;F2 决定是否需要新增 force-flag knob)|

#### 4.9.2 F1 不设计场景目录

F1 仅暴露 knob shape;**具体哪些 knob 组合构成"GPS 失效"故障、持续多久、注入时机**留给 [H4](../H-verification/H4-fault-catalog.md)。H4 在 Wave 11 启动时 forward-consume F1 §4.4.1 knob 表;若 H4 发现需要 F1 未暴露的 knob(例如"position_valid 强制 0 而 gps_healthy=true"的怪异组合),F1 通过变更日志补 knob。

#### 4.9.3 与 [H1 验证场景目录](../H-verification/H1-scenario-catalog.md)(Wave 11)的耦合

H1 与 H4 同 wave、同 batch(per [01-design-relationships.md §5](../01-design-relationships.md) co-seal pair `(H1, H4)`)。H1 中的 noisy stress 场景同样消费 F1 knob;F1 同样不设计 H1 场景内容,仅承诺 knob 可用。

### 4.10 越权显式排除清单(Out-of-scope)

为避免 F1 越权设计,本节显式列出 F1 **不**做但容易被误认为应做的设计点:

| 越权项 | 实际归属 | 引用 |
|---|---|---|
| 真实 INS 估计算法(EKF / 互补滤波 / 多传感器融合)| firmware-owned;**不**在模型仓 | 架构 v1 §3 / §12.3 |
| ins_stub 内部块级结构(子系统拆分、`Plant_States_Bus → INS_Out_Bus` 字段映射的实现拓扑、变体切换的 Simulink 块选择)| F2 | [F2 ins_stub 结构设计](F2-ins-stub-structural.md) |
| FMS / Controller 对 `INS_Out_Bus` 字段的依赖矩阵、validity 处理、fallback 行为 | F3 | [F3 INS_Out_Bus 消费规则](F3-consumption-rules.md);F1 仅在 §4.7.3 引用 |
| 多旋翼 / 固定翼 / VTOL 的具体 σ / bias / delay **数值**默认表 | B3 PLANT_PARAM 或 harness-local;H4 / H1 场景表 | [B3](../B-contracts/B3-parameter-schema.md) / [H1](../H-verification/H1-scenario-catalog.md) / [H4](../H-verification/H4-fault-catalog.md) |
| `INS_Out_Bus` 字段顺序 / 字节布局 / 类型 | B1 + B5 mirror artifact | [B1 §4.3.1](../B-contracts/B1-bus-inventory.md) / [B5 §4.5](../B-contracts/B5-ins-bus-mirror.md) |
| `INS_Status` / `INS_Flag` 位号 / 数值 | B2 | [B2 §4.4.10 / §4.4.11](../B-contracts/B2-enum-inventory.md) |
| firmware → 模型仓 mirror PR 工作流、抽取脚本、版本标记、drift 检测 | B5 + I2 + I3 | [B5](../B-contracts/B5-ins-bus-mirror.md) / [I2](../I-tooling/I2-bus-enum-mirror.md) / [I3](../I-tooling/I3-contract-diff.md) |
| H4 故障场景具体目录、触发条件、持续时长 | H4 | [H4](../H-verification/H4-fault-catalog.md);F1 仅暴露 knob |
| H3 RNG seed 矩阵管理、批量回归 seed 分配 | H3 | [H3](../H-verification/H3-regression-baseline.md);F1 仅承诺 ins_stub 接受 seed |
| Plant 内部传感器物理仿真(IMU / MAG / BARO / GPS / Airspeed bus)| C1 + C2 + C3 | [C1](../C-plant/C1-plant-functional.md) / [C2](../C-plant/C2-plant-structural.md) / [C3](../C-plant/C3-multicopter-leaf.md);ins_stub **不**消费这些 sensor bus,只走 `Plant_States_Bus` truth |
| harness 顶层 ins_stub 接入拓扑、ideal vs noisy 变体的 harness-side 选择载体 | G1 | [G1 MIL 顶层结构](../G-harness/G1-mil-toplevel.md) |
| ins_stub 输出在 G3 logsout 中的信号选择 | G3 | [G3 日志与可观测性](../G-harness/G3-logging.md) |

## 5. 已知风险与悬而未决问题

- **firmware commit hash 暂缺**
  - 影响:RULES §5 / §3 表 require 镜像自 firmware 的契约记录 commit hash + 文件相对路径;本文件 §3 firmware 引用仅给路径占位 `FMT-Firmware @ <pending hash>`。注:F1 自身 contract_impact=no 且**不**直接镜像 firmware(mirror 由 [B5](../B-contracts/B5-ins-bus-mirror.md) 落地);F1 只在 §3 引用 firmware 头作为上游参考。
  - 处置:open;评审通过前由 reviewer / orchestrator 在 INDEX 决策日志登记本工作项时补齐(per A3 / A6 / A7 / B1 / B2 / B5 同 batch 处置约定)。

- **C1 同期 sibling 仅 draft 状态**
  - 影响:F1 §4.5 字段映射的 `Plant_States_Bus` truth 来源依赖 C1 锁定的 Plant 输出能力;C1 当前 status=draft(Wave 6 in flight),per 任务说明"may forward-cite, NOT formal co-seal"。F1 当前以 [A3 §4.3.5](../A-architecture/A3-module-boundaries.md) 已 reviewed 字段表为权威。
  - 处置:open;若 C1 reviewed 时收紧某些字段(例如 Plant 不输出 `acc_b_mps2`),F1 §4.5 字段映射通过变更日志同步,带扰版相应通道 knob 标 N/A。

- **firmware INS_Out_Bus 实际是否含独立 mag / baro / airspeed 字段未确认**
  - 影响:§4.5.1 给"firmware INS_Out_Bus 通常不直接呈现 mag / baro / airspeed 字段"的 best-effort 假设;若 mirror artifact 显示 firmware 实际含独立字段,F1 §4.5 字段表与 §4.4.1 knob 表均需扩展。
  - 处置:open;待 [B5](../B-contracts/B5-ins-bus-mirror.md) mirror artifact 首次落地后通过变更日志同步;[A3 §4.4.4](../A-architecture/A3-module-boundaries.md) 当前枚举的 7 个 navigation 字段是 F1 当前覆盖范围。

- **理想 vs 带扰变体的切换载体未定**
  - 影响:§4.1.3 决议 "不经 [A5](../A-architecture/A5-variant-strategy.md) Variant Subsystem,而是 harness-side 参数选择";具体载体(布尔 flag / enum knob / Simulink mask param / harness YAML / Variant Source 块)由 [F2](F2-ins-stub-structural.md) + [G1](../G-harness/G1-mil-toplevel.md) 决定。F1 仅承诺"二选一可切换且互不污染"。
  - 处置:已规避(F1 不锁载体形式,F2 / G1 共同决定);若 F2 选择某种载体后发现需要 F1 介入(例如需要新增 `variant` 之外的 knob),F1 通过变更日志同步。

- **bias_walk reset 行为是 zero 还是 preserve**
  - 影响:§4.3.3 / §4.7.2 默认 reset 时 `bias_walk → 0`;但实际 firmware 估计器 bias 状态在 reset 时可能保留(EKF state 部分保留)。F1 当前选 zero(更接近 ideal 退化语义,且复现性更好);若 H4 / SIH 对比显示需要 preserve,F1 修订。
  - 处置:open;待 SIH 对比阶段(Phase 3)校准;F1 当前默认 zero 与 [A6 §4.7.1](../A-architecture/A6-init-reset-contract.md) "reset / 故障注入恢复" 表的 "RNG / 噪声 / 偏置:ins_stub / 故障注入(H4)有" 一致。

- **GPS lock latency 中 `position_lla` 字段在未 lock 期间的输出值(zero vs NaN vs hold-last)未锁定**
  - 影响:§4.3.7 / §4.5 表给"未 lock 期间 LLA → zero/NaN(具体由 F2 决定)";不同选择对消费侧 fallback 行为有不同诱发(F3 范围)。
  - 处置:open;由 [F2](F2-ins-stub-structural.md) + [F3](F3-consumption-rules.md) 共同决定输出值与 fallback 配对;F1 锁住"未 lock 期间 `gps_valid = 0` + LLA 不可信"的契约。

- **`INS_Status` enum vs bitfield 形态(继承自 [B2 §5](../B-contracts/B2-enum-inventory.md))**
  - 影响:[B2 §4.4.10](../B-contracts/B2-enum-inventory.md) 按 enum 锁;[B1 §4.3.1](../B-contracts/B1-bus-inventory.md) 字段类型按 `uint32` bitfield;首次 [B5](../B-contracts/B5-ins-bus-mirror.md) mirror artifact 落地若揭示 firmware 实际是 bitfield,F1 §4.5 表 `INS_Status` 行的"立即 READY"语义改成"set ready bit",但 F1 锁定的"理想版立即 ready / 带扰版 transient"行为本身**不**变。
  - 处置:open;由首次 mirror PR 决定,F1 通过变更日志同步。

- **knob 数量增长导致 PARAM 表膨胀风险**
  - 影响:§4.4.1 knob 表暴露 ~13 个 knob 类目,per-field 展开后总 knob 数 ≥ 50。若纳入 [B3 PLANT_PARAM](../B-contracts/B3-parameter-schema.md) 会显著增大 firmware-side `PLANT_PARAM` 字节宽度。
  - 处置:open;**建议** B3 把 ins_stub knob 集合放在 **harness-local** PARAM 而非 `PLANT_PARAM`(harness PARAM 不参与 firmware 契约 diff,因 ins_stub 本就 harness-only);具体由 B3 在自身退出条件中决定;F1 不强制路径,只承诺"runtime-tunable"。

- **knob 单位与 [A2 命名约定](../A-architecture/A2-naming-conventions.md) 的对齐**
  - 影响:§4.4.1 表 "单位" 列与 §4.4.3 命名建议;A2 R-10 系列对单位后缀有约束(`_m`, `_mps`, `_radps`);F1 knob 路径建议 `ins_stub.sigma_noise.pos_ned_m`(σ 的单位继承被噪声字段单位)。
  - 处置:已规避(F1 §4.4.3 给 A2 命名规范的对齐建议;A2 / B3 决定是否采纳)。

- **本工作项 contract_impact=no 的影响范围**
  - 影响:F1 是 harness-only fixture 的功能设计;不触及 firmware 契约;ins_stub 不参与 [B4 契约 diff](../B-contracts/B4-contract-diff.md) 与 [I3 contract diff](../I-tooling/I3-contract-diff.md);ins_stub 不在 `export/firmware/` 下出现。
  - 处置:已声明 contract_impact=no;Self-check item 6(INDEX 决策日志登记契约影响)N/A;item 7(镜像自 firmware 的契约 commit hash)N/A — F1 不**直接**镜像 firmware,只通过 [B5](../B-contracts/B5-ins-bus-mirror.md) mirror artifact 间接消费 schema(B5 自身 contract_impact=yes 与 mirror commit hash 已登记)。

## 6. 退出条件复核

F1 在 [`00-design-plan.md §4.F`](../00-design-plan.md) 中的退出条件原文:

> 理想版/带扰版的输出特性、参数化(噪声/偏差/延迟)范围

逐条复核(任务说明把退出条件分解为 (a) 理想版输出特性、(b) 带扰版输出特性、(c) 参数化 knob 形态三项):

| # | 退出条件原文(分解)| 本文档依据 | 状态 |
|---|---|---|---|
| (a) | 理想版输出特性 | §4.2 全节(§4.2.1 通用约束 identity transform;§4.2.2 validity 立即 valid;§4.2.3 timestamp;§4.2.4 适用场景);§4.5 字段映射表"理想版变换"列 | 满足 |
| (b) | 带扰版输出特性 | §4.3 全节(§4.3.2 噪声;§4.3.3 偏差 const + walk;§4.3.4 transport delay;§4.3.5 validity 启动 transient;§4.3.6 jitter;§4.3.7 GPS lock latency;§4.3.8 sensor health 开关;§4.3.9 适用场景);§4.5 字段映射表"带扰版叠加"列 | 满足 |
| (c) | 参数化(噪声 / 偏差 / 延迟)范围 | §4.4 全节(§4.4.1 knob 总表 13 类含 σ_noise / bias_const / σ_walk / delay_samples;§4.4.2 数值范围与 shape 边界声明;§4.4.3 命名约定);F1 锁 **shape**(每 knob 类型 / 单位 / 应用通道 / 默认行为类别),具体数值范围 forward-defer 到 H1 / H4 / B3 — 这是 F1 范围与 H4 故障目录的明确分工 | 满足 |

附:对**审计 [F-04](../_audit/2026-05-05-plan-audit.md) 的 widening 要求**("Widen F1 noise/bias/delay knobs"):§4.4.1 knob 表的 σ_noise + bias_const + σ_walk + delay_samples 四个 navigation knob 类 + 5 个 transient/jitter/health knob 标量类共 13 类;覆盖审计 F-04 demand 的"noise / bias / delay knob widening"。审计 F-04 的另一条路径(H4 fault catalog)由 [H4](../H-verification/H4-fault-catalog.md) 在 Wave 11 落地,二者互补(F1 提供 knob shape,H4 提供 knob 值矩阵)。

附:对**审计 [F-05](../_audit/2026-05-05-plan-audit.md) RNG seeding** 的 forward-cite:§4.6 全节锁定 ins_stub 不持 RNG state、外部 seed 注入、deterministic 复现承诺;具体 seed 矩阵管理由 [H3](../H-verification/H3-regression-baseline.md) 在 Wave 11 落地。

附:对**架构 v1 §17 risk 4 INS-stub realism risk** 的 mitigation 闭合:架构 v1 mitigation 文本 = "提供 noise/bias/delay 可配 knob,要求至少一个 MIL 场景在带扰下跑";F1 §4.3 + §4.4 提供 knob;"至少一个带扰场景"由 [H1](../H-verification/H1-scenario-catalog.md) 落地。

退出条件全部满足;状态待 reviewer 升级到 reviewed。

## 7. 下游影响

按 [`01-design-relationships.md §4.6 + §4.10`](../01-design-relationships.md) F1 出边:

```text
B5 → F1                    (上游;§3 / §4.5 / §4.8 引用)
C1 ⇢ F1                    (上游;§3 / §4.5 引用,sibling forward-cite)
F1 → F2                    (下游;F2 在结构上承接 F1 功能契约)
H4 ⇢ F1                    (下游;§4.9 H4 forward-consume F1 knob shape)
```

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [F2 ins_stub 结构设计](F2-ins-stub-structural.md) *(未启动,Wave 7)* | → | §4.1.3 两变体集合(F2 决定结构上如何实现切换);§4.2 / §4.3 输出行为(F2 决定块级实现);§4.4.1 knob 集合(F2 决定 knob 在 Simulink 上的载体 — mask param vs PARAM struct vs harness YAML);§4.5 字段映射(F2 决定每行的 Simulink 块拓扑);§4.6 RNG seed 派生(F2 决定 sub-seed 函数);§4.7 init/reset 实现 |
| [F3 INS_Out_Bus 消费规则](F3-consumption-rules.md) *(未启动,Wave 9)* | (transitive via shared `INS_Out_Bus` schema;无直接 F1→F3 边,但 F3 设计需了解 ins_stub 在带扰下哪些位会归零、哪些字段会失效以设计消费侧 fallback)| §4.3.5 validity transient timeline(让 F3 知道 reset 后 N ms 内某些位会是 0);§4.3.7 GPS lock latency 期间 LLA 不可用;§4.3.8 sensor health 开关 → INS_Flag.* 位归零模式;F3 据此设计每字段 fallback |
| [H4 故障注入目录设计](../H-verification/H4-fault-catalog.md) *(未启动,Wave 11)* | ⇢ | §4.4.1 knob shape 表是 H4 故障场景的最低层 actuator;§4.9.1 H4 故障类与 F1 knob 设置的典型映射;H4 在 Wave 11 启动时 forward-consume F1 knob 表 |
| [H1 验证场景目录设计](../H-verification/H1-scenario-catalog.md) *(未启动,Wave 11,与 H4 同 batch)* | ⇢(经 H4)| §4.4.1 knob shape;§4.3 带扰版行为(让 H1 noisy stress 场景设计有依据);架构 v1 §17 risk 4 mitigation"至少一个带扰场景"在 H1 落地 |
| [H3 回归基线策略设计](../H-verification/H3-regression-baseline.md) *(未启动,Wave 11)* | ⇢(经 §4.6 RNG seeding 契约 forward-cite)| §4.6 全节(ins_stub 不持 RNG state、外部 seed 注入、deterministic 复现承诺);H3 落 seed 矩阵管理时承接 |
| [G1 MIL 顶层结构设计](../G-harness/G1-mil-toplevel.md) *(未启动,Wave 10)* | ⇢ | §4.1.2 ins_stub harness-only boundary;§4.1.3 变体选择载体留给 G1 + F2 共同决定;ins_stub 接入位置(架构 v1 §11 路径) |
| [G3 日志与可观测性设计](../G-harness/G3-logging.md) *(未启动,Wave 10)* | ⇢ | §4.5 字段表(让 G3 知道哪些 INS_Out_Bus 字段值得 log);§4.3.5 validity timeline(让 G3 知道哪些 INS_Flag 位的 toggling 值得记录);§4.3.6 jitter knob(让 G3 知道 timestamp 字段可能含 jitter)|
| [B3 Parameter schema 设计](../B-contracts/B3-parameter-schema.md) *(已 reviewed,但 ins_stub knob 是否纳入 PLANT_PARAM 由 B3 决定)* | ⇢(soft;§5 风险列出建议)| §4.4.1 knob 集合数量 ≥ 50(per-field 展开后);§5 已建议 B3 把 ins_stub knob 放 harness-local 而非 PLANT_PARAM;B3 视情况采纳 |
| [I1 init 脚本设计](../I-tooling/I1-init-script.md) *(未启动,Wave 12)* | ⇢ | §4.4 knob 注入路径(I1 加载 PLANT_PARAM 或 harness-local 配置时把 knob 值带到 ins_stub workspace);§4.6 RNG seed 注入(I1 把 seed 从场景文件加载到 base workspace) |
| [I5 sim runner 脚本设计](../I-tooling/I5-sim-runner.md) *(未启动,Wave 12)* | ⇢ | §4.6 RNG seed CLI 接口(I5 批量回归 / 单跑时把 seed 传递到 ins_stub);§4.4 variant 选择 CLI |

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-08 | F1 author | 初稿;锁定 ins_stub harness-only boundary、ideal / noisy 两变体集合、13 类 knob shape;forward-cite [H3](../H-verification/H3-regression-baseline.md) RNG seed contract、[H4](../H-verification/H4-fault-catalog.md) fault catalog;承接 [A6 §4.7.1](../A-architecture/A6-init-reset-contract.md) 对 F1 的 explicit demand("ins_stub 必须模拟 INS_Status / INS_Flag 位域,使 FMS 同样 gate")+ 架构 v1 §17 risk 4 mitigation;消费 [B5](../B-contracts/B5-ins-bus-mirror.md) mirror artifact 作为输出 schema 真理;对 [C1](../C-plant/C1-plant-functional.md) 同 wave 6 sibling 做 forward-cite(非 co-seal)。commit hash 占位 `<pending hash>` 与 A3 / A6 / A7 / B1 / B2 / B5 同 batch 模式一致。 |

## Self-check

- [x] frontmatter 完整,字段值合法
- [x] 上游文档全部存在且 status ≥ reviewed,**或**为本工作项所在 co-seal batch 的同批兄弟(架构 v1 已发布;A1 / A3 / A4 / A5 / A6 / A7 / A8 全部 reviewed;B1 / B2 / B5 全部 reviewed 2026-05-07;Wave 6 同期 sibling C1 仍 draft,本工作项**不与**其 co-seal batch 但允许前向引用 — 引用位置已在 §3 / §4.5 / §5 标 "drafting in flight" / "C1 同期 sibling 仅 draft" / "forward-cite",per 任务说明 "may forward-cite, NOT formal co-seal" 与 RULES §6 self-check 第 2 项"co-seal sibling"条款语义最佳近似)
- [x] 退出条件逐条复核完成,每条均给出依据(§6 表 3 项 + 审计 F-04 widening 闭合 + 审计 F-05 RNG seeding 闭合 + 架构 v1 §17 risk 4 mitigation 闭合三项附录)
- [x] 引用路径全部可点击访问(本文件所有引用均使用 RULES §4 规定的相对路径形式;Wave 6 sibling C1 + Wave 7+ 下游 F2 / F3 / G1 / G3 / H1 / H3 / H4 / I1 / I5 文件在后续 wave 创建,符合 sibling-or-pending 约定)
- [x] 不存在 RULES §5 禁则中的内容(无 .slx 截图;无可执行 .m 代码;§4.3.2 / §4.3.3 / §4.3.6 数学表达式是 schema 描述非脚本;§4.6.4 伪代码用 `pseudo` 标识仅作意图说明 — 实际本文件未含 pseudo 块,只含数学公式;不复述 firmware 实现细节;不重定义 INS_Out_Bus 字段表 — 字段表由 [B1 §4.3.1](../B-contracts/B1-bus-inventory.md) + [B5](../B-contracts/B5-ins-bus-mirror.md) mirror artifact 锁定,本文件 §4.5 只回引并标"以 mirror 为准";不重定义 `INS_Status` / `INS_Flag` 数值 — 引用 [B2 §4.4.10 / §4.4.11](../B-contracts/B2-enum-inventory.md);FMT-Firmware 引用以路径占位 + `<pending hash>` per A3/A6/A7/B1/B2/B5 batch 模式)
- [N/A] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记 — F1 contract_impact=**no**(harness-only fixture;不触 firmware 可见 bus / enum / parameter / EXPORT;ins_stub 不导出 firmware,不参与 B4 / I3 契约 diff);此项不适用
- [N/A] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径 — F1 **不**直接镜像 firmware(mirror 由 [B5](../B-contracts/B5-ins-bus-mirror.md) 落地;F1 通过 mirror artifact 间接消费 schema,B5 自身已记录 commit hash);此项不适用
- [x] 下游影响已沿关系图识别完毕(§7 已覆盖 [01-design-relationships.md §4.6 + §4.10](../01-design-relationships.md) 全部 F1 出边:`F1 → F2` 与 `H4 ⇢ F1`;并扩展含 transitive F3 / 后 wave G1 / G3 / H1 / H3 / B3 / I1 / I5 的承接关系)
- [x] 文档不超出本工作项范围(无越权设计;§4.10 显式列出 11 类 out-of-scope 项与对应归属:F2 块级结构 / F3 消费规则 / B1 + B5 字段表 / B2 enum 数值 / B3 数值默认 / H4 故障场景 / H3 seed 矩阵 / C1 + C2 + C3 sensor 物理仿真 / G1 harness 接入 / G3 logging 信号选择 / firmware INS 算法)
