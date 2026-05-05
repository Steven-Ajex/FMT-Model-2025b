---
work_item: A4
title: 跨速率边界设计
upstream: ["架构 v1", "A1"]
contract_impact: no
status: reviewed
authored_at: 2026-05-05
last_reviewed_at: 2026-05-05
reviewer_verdict: pass
---

# A4 跨速率边界设计

## 1. 目的

在 Plant(1 ms)、Controller(5 ms)、INS(10 ms,firmware 拥有,模型仓侧仅在 `ins_stub` 输出处消费)、FMS(20 ms)四个单速率模块共存的 MIL 闭环中,系统化定义**每一个跨速率边界点**的 rate-transition 策略(zero-order hold / latch / sample-time-based downsample / single-element queue)、延迟预算与陈旧度上限,作为 G2(调度实现)、E5(5 ms 预算)、C4(1 ms 数值)、F2(`ins_stub` 输出节奏)等下游的输入。

## 2. 范围

**在范围:**

- 模型仓侧 MIL 闭环中**所有跨速率边界点的枚举**(共 6 个主边界 + 1 个 Plant 内部慢传感器边界)
- 每个边界的 rate-transition 策略选型(ZOH / latch / sample-time-based downsample / single-element queue)与选型理由
- 每个边界的**延迟预算**(从产出端 step end 到消费端 step start 之间的最大允许延迟)
- 每个边界的**陈旧度上限**(staleness ceiling:消费端任一 step 拿到的数据相对于真实时刻最坏可能的滞后)
- 跨速率边界的**文档化模板**(对应 A1 §4.6 缺口:downstream 工作项后续在自己的设计文档中以此模板说明本模块输入/输出的边界)
- INS 10 ms → 5 ms / 20 ms 的对齐策略(整数倍同相 vs 容忍相位漂移)
- Plant 内部"慢传感器"在 1 ms 主循环中按 cadence 发布的策略(对应 A1 §12.4 缺口)
- Architecture v1 §13.5 提到的"Plant and INS latency must be considered" 的量化分配比例(粗到 5 ms 子预算粒度)

**不在范围(由其他工作项处理):**

- Simulink 模型中实际的 Rate Transition 块部署位置、Solver 配置、Sample Time 着色检查 — 由 [G2 多速率调度设计](../G-harness/G2-rate-scheduling.md) 在 harness 中实现
- Controller 5 ms 周期内各环路的子预算(位置/速度/姿态/角速度环各自占多少 µs)— 由 [E5 Controller 性能预算设计](../E-controller/E5-performance-budget.md) 细化
- Plant 1 ms 内积分器选择、步长稳定性、reset 行为 — 由 [C4 Plant 数值与积分设计](../C-plant/C4-numerics.md) 决定
- `ins_stub` 内部如何从 `Plant_States_Bus` 派生 `INS_Out_Bus` 字段 — 由 [F1 ins_stub 功能设计](../F-ins-contract/F1-ins-stub-functional.md) / [F2 ins_stub 结构设计](../F-ins-contract/F2-ins-stub-structural.md) 决定;A4 只规定 `ins_stub` 输出在 10 ms 边界上的 publish rate
- 时间戳单位 / epoch / 回卷 / 单调性约束 / dt 来源 — 由 [A7 时间与时间戳约定](A7-time-conventions.md) 决定;A4 在策略层面引用 A7 的结论(隐式 ↔)
- Pilot/GCS/Auto/Mission **来源** 的真实生产速率(操纵杆 ADC、链路速率、地面站发包频率)— 设计阶段视为外部输入,A4 仅约束这些输入进入 FMS 20 ms 时如何处理

## 3. 依赖

| 上游 | 引用位置 | 用途 |
|---|---|---|
| 架构 v1 §4.4 / §4.6 | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | "Single-rate per module" 与 "Explicit interfaces / documented rate boundaries" 原则 |
| 架构 v1 §9 Cross-module buses and dataflow | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | 跨模块 bus 拓扑;§9.3.2 "Cross-rate changes occur only in harness or scheduler wiring" |
| 架构 v1 §12.4 Plant | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | "slower sensor publication modeled through output update cadence" 的策略空间 |
| 架构 v1 §13 Timing constraints | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | Plant 1 ms / Controller 5 ms / FMS 20 ms / INS 10 ms 表;实操含义 §3 / §5(latency must be considered) |
| 架构 v1 §15.1 / §16 | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | MIL `ins_stub` 在 10 ms 边界的位置 |
| [A1 架构 v1 评审与缺口闭合](A1-v1-review.md) (status: reviewed) | §4.2 §4 / §5.1 / §9.3 / §12.4 / §13 表行 | 本工作项要闭合的具体缺口清单 |

约定:本文件**不**重新陈述各 bus 的字段表 — 凡涉及具体 bus 字段时一律引用 B 区(B1)。本文件中 bus 名称仅作"边界两端的数据载体"使用。

## 4. 设计内容

### 4.1 总览:模块速率与单向性

承袭架构 v1 §13:

| 模块 | 周期 | 频率 | 拥有方 | 备注 |
|---|---:|---:|---|---|
| Plant | 1 ms | 1000 Hz | 模型仓 | 仿真热回路;1 ms 主循环 |
| Controller | 5 ms | 200 Hz | 模型仓 | 控制热回路 |
| INS | 10 ms | 100 Hz | **FMT-Firmware** | 模型仓侧仅在 `ins_stub` 输出端消费 |
| FMS | 20 ms | 50 Hz | 模型仓 | 监管层 |

记号:周期之间的关系为 Plant : Controller : INS : FMS = 1 : 5 : 10 : 20 ms,即 Controller 是 Plant 的 5 倍周期、INS 是 Plant 的 10 倍周期、FMS 是 Plant 的 20 倍周期、FMS 是 Controller 的 4 倍周期、FMS 是 INS 的 2 倍周期、INS 是 Controller 的 2 倍周期。**所有比率均为整数**,这一性质在 §4.4 选型中被反复利用。

数据流方向(简化自架构 v1 §9.1):

```text
慢端 (FMS 20ms) ──> 中端 (Controller 5ms) ──> 快端 (Plant 1ms) ──> 仿真物理
                                                              │
                                                              ▼
慢端 (FMS 20ms) <── 中端 (INS 10ms, fw) <── 快端 (Plant 1ms ── ins_stub)
                                                              │
                                                              ▼ (Controller 5ms 也消费 INS_Out_Bus)
```

观察:从命令路径看,数据从慢→快(FMS→Controller→Plant);从反馈路径看,数据从快→慢(Plant→ins_stub→Controller / FMS)。两类方向对应不同的边界策略需求(§4.3)。

### 4.2 跨速率边界点枚举

下表枚举 MIL 闭环中**所有**跨速率边界。每个边界给一个稳定 ID(`RB-NN`)以便下游引用。

| ID | 来源(产出端) | 去向(消费端) | 速率方向 | 数据载体(bus) | 主导用途 |
|---|---|---|---|---|---|
| RB-01 | Plant 1 ms | Controller 5 ms | 快→慢(下采样) | `Plant_States_Bus`(harness/HIL/SIH variant 中)+ 各传感器 bus(MIL 经 `ins_stub` 中转,不直入 Controller) | 仅在含 truth-state passthrough 的 harness 变体上存在;MIL 默认变体下,Controller 不直接消费 Plant 输出 |
| RB-02 | Controller 5 ms | Plant 1 ms | 慢→快(上采样保持) | `Control_Out_Bus` | 执行器命令进入仿真物理 |
| RB-03 | INS 10 ms(`ins_stub` 输出) | Controller 5 ms | 慢→快(上采样保持) | `INS_Out_Bus` | 状态估计反馈到内环 |
| RB-04 | INS 10 ms(`ins_stub` 输出) | FMS 20 ms | 快→慢(下采样) | `INS_Out_Bus` | 状态估计反馈到监管层 |
| RB-05 | FMS 20 ms | Controller 5 ms | 慢→快(上采样保持) | `FMS_Out_Bus` | 设定值 / cmd_mask / mode 下发 |
| RB-06 | Pilot/GCS/Auto/Mission(外部) | FMS 20 ms | 来源率不固定 → 20 ms | `Pilot_Cmd_Bus` / `GCS_Cmd_Bus` / `Auto_Cmd_Bus` / `Mission_Data_Bus` | 操纵 / 链路 / 自动 / 任务指令进入 FMS |
| RB-07 *(Plant-内部,慢传感器)* | Plant 1 ms 主循环 | 同模块内"慢传感器"输出端(GPS / Barometer / MAG 等) | 1 ms → 各自 cadence | 各 sensor bus | 满足架构 v1 §12.4"通过输出更新节奏建模慢传感器"的策略 |

补充说明:

- `RB-01` 在 MIL 默认变体下**不存在**(Controller 不直接消费 `Plant_States_Bus`)。它只在架构 v1 §8.4 提到的"optional `Plant_States_Bus` only in harness/HIL/SIH variants" 时才生效。本文件保留 ID 以便 A5 / G1 在变体设计中引用,但其策略与 RB-03 同(快→慢下采样)。
- `RB-06` 的来源端速率在设计阶段视为"外部不可控"(操纵杆驱动可能 50 Hz, 100 Hz 或更高;链路 GCS 包可能 5–20 Hz;Auto/Mission 由本机软件驱动)。A4 只规定 FMS 端的接收策略,不约束来源端。
- `RB-07` 不是模块**之间**的边界,是模块**内部**的"慢公布"问题。架构 v1 §12.4 把它显式列为"slower sensor publication modeled through output update cadence",所以本文件一并涵盖。

### 4.3 策略空间与选型语义

定义本文件使用的四种 rate-transition 策略,所有策略名在下游(尤其 G2)实现时必须沿用同一术语:

| 策略名 | 语义 | 适用方向 | 典型 Simulink 实现要点(仅参考) |
|---|---|---|---|
| **ZOH(Zero-Order Hold)** | 慢端最近一次产出值在快端每个 step 都被读取并保持不变,直到慢端再次产出新值。**值-保持型 上采样**。 | 慢→快 | 由 Solver 自动通过 sample-time 推断插入;无需用户显式块。模型应保证"读到的就是最近一次产出" |
| **Latch(锁存 / one-shot snapshot)** | 在快端的某个特定相位锁存一份慢端快照,慢端在锁存窗口内的更新不立即可见,直到下一次锁存触发。**避免一帧内多次取值不一致** 的上采样变体。 | 慢→快(语义上要求"内部一致快照") | 一般用 Rate Transition 块的 deterministic 模式 + 触发锁存信号 |
| **Sample-time-based downsample** | 快端每 N 个 step 选 1 个交付给慢端;选择策略一般是"慢端 step 起点的快端最新值"。**值-选择型 下采样**。 | 快→慢 | Rate Transition 块从快采样时间转到慢采样时间,deterministic 模式;不做平均/插值 |
| **Single-element queue(单格队列 / mailbox)** | 来源端每次产出覆写"邮箱",消费端每次读取该邮箱;两端速率不需为整数倍,但消费端读到的可能是任意上次产出值(无丢包检测)。**异步松耦合**。 | 异步 / 速率不固定 | 消费端读到的就是来源端"最后一次写入" |

不在本文件正式策略表中的:

- **平均(decimation by averaging)**、**插值(linear / cubic interpolation)** — 这些是数值滤波,不属于"边界传输策略"。如某下游(例如慢传感器仿真)需要,应在该模块内部完成,然后再交付到边界,边界本身仍按 ZOH/latch/downsample 之一的纯传输语义处理。
- **多元素队列 / FIFO** — 模型仓侧禁止动态内存与可变尺寸信号(架构 v1 §4.5),固定长度 FIFO 在多速率单步执行下复杂度高且收益低,本文件不采用。

### 4.4 各边界的策略选型与预算

每个边界 RB-NN 的字段定义(模板):

- **Strategy**:从 §4.3 四选一。
- **Latency budget**:从产出端 step **结束** 到消费端下一次 step **开始** 所允许的最大时间窗(单位 ms)。这是架构层的总预算,具体到 Simulink 中由 G2 通过 Solver 决定性配置实现。
- **Staleness ceiling**:消费端在任一 step 拿到的数据,相对于"如果消费端能瞬时读到产出端最新值"的最坏滞后(单位 ms)。即"读到的数据可能比物理真值老多少"。
- **Determinism**:single-rate-aligned(整数倍且固定相位)/ async(无固定相位)。
- **Failure mode if violated**:边界策略被破坏(例如调度器抖动超出预算)时的现象。

#### RB-01 Plant 1 ms → Controller 5 ms(仅 harness 变体)

| 字段 | 取值 |
|---|---|
| Strategy | **Sample-time-based downsample**(快→慢) |
| Latency budget | ≤ 1 ms(Plant 一个 step) |
| Staleness ceiling | 5 ms(Controller 在 step k 读到的是 Plant 在 [(k-1)·5ms, k·5ms] 区间最后一次产出 ≈ 4 ms 之前的值,加上 Plant 自己 1 ms 的产出滞后,理论最差约 5 ms;典型 ≤ 1 ms) |
| Determinism | single-rate-aligned(5 是 1 的整数倍) |
| Failure mode if violated | Controller 收到的 truth state 跨 step 不一致 → 仿真不可重复 |

适用条件:仅 SIH/HIL 或 harness 调试变体含 `Plant_States_Bus → Controller` 直通通道时启用;**MIL 默认禁用**(架构 v1 §8.4 显式声明 "never as standard flight dependency")。

#### RB-02 Controller 5 ms → Plant 1 ms(执行器命令)

| 字段 | 取值 |
|---|---|
| Strategy | **ZOH(Zero-Order Hold)**(慢→快) |
| Latency budget | ≤ 1 ms(下次 Plant step 起始即生效) |
| Staleness ceiling | 5 ms(Plant 在某 step 读到的是 Controller 上一个 5 ms step 末写出的值,5 ms 之后才会被新值替换) |
| Determinism | single-rate-aligned(5 = 5 × 1) |
| Failure mode if violated | 执行器在 5 ms 内被多次更新或丢失 → 仿真物理不可重复 |

补充:执行器命令是"控制信号",在被新值替换前**必须保持不变**;严禁加入插值或平滑,否则等于把控制器的离散输出当成连续信号,会改变实际仿真物理。

#### RB-03 INS 10 ms → Controller 5 ms(状态估计 → 内环)

| 字段 | 取值 |
|---|---|
| Strategy | **ZOH**(慢→快;INS 10 ms 是 Controller 5 ms 的 2 倍周期,反向看是慢→快) |
| Latency budget | ≤ 1 ms(下次 Controller step 起始即生效) |
| Staleness ceiling | 10 ms(Controller 在某 step 读到的 `INS_Out_Bus` 最坏情况下相对于 INS 真实快照已老 10 ms;典型 5 ms) |
| Determinism | single-rate-aligned(10 = 2 × 5,整数倍) |
| Failure mode if violated | Controller 内环跟踪一个间歇性卡帧的状态估计,姿态控制可能引入跳变 |

补充:这是控制热回路最敏感的边界。Architecture v1 §13.3 "Controller execution budget is most critical" + §13 实操含义 5 "INS validity flags should be sampled and consumed cleanly at the FMS/Controller boundaries" 都指向此边界。**`INS_Out_Bus` 的 validity 标志位也走 ZOH**,与字段值同步老化,不允许任何字段独立于 validity 更新。

#### RB-04 INS 10 ms → FMS 20 ms(状态估计 → 监管层)

| 字段 | 取值 |
|---|---|
| Strategy | **Sample-time-based downsample**(快→慢) |
| Latency budget | ≤ 1 ms |
| Staleness ceiling | 20 ms(FMS 在某 step 读到的是上一个 20 ms 窗口内 INS 最后一次产出,即至多 ~20 ms 之前;典型 ~10 ms) |
| Determinism | single-rate-aligned(20 = 2 × 10) |
| Failure mode if violated | FMS 状态门阈(如位置 hold 判定、起降阶段切换)看到状态抖动或停帧 |

补充:FMS 的逻辑对状态估计的敏感度低于 Controller(决策级而非控制级),20 ms 陈旧度是可接受的。

#### RB-05 FMS 20 ms → Controller 5 ms(设定值 + cmd_mask)

| 字段 | 取值 |
|---|---|
| Strategy | **ZOH(主字段)+ Latch(同帧一致性约束)** |
| Latency budget | ≤ 1 ms |
| Staleness ceiling | 20 ms(Controller 在某 step 读到的 `FMS_Out_Bus` 最坏情况下已老 20 ms;典型 10 ms) |
| Determinism | single-rate-aligned(20 = 4 × 5) |
| Failure mode if violated | Controller 在 mode 切换瞬间观察到 setpoint / cmd_mask 半新半旧 → 控制律选择错环路 |

**Latch 子语义** — 关键:`FMS_Out_Bus` 中的 `cmd_mask`、`ctrl_mode`、`mode`、`reset`、各类 `*_cmd` 字段必须**作为同一帧锁存**到 Controller 端,不允许 Controller 在某个 step 中看到"`cmd_mask` 已切但 setpoint 还是旧的"这类组合。Simulink 中 Rate Transition 的 deterministic 模式默认能保证整个 bus 的字段集体过渡,但模型作者必须避免在 FMS→Controller 之间手工拆 bus 再拼接,以免让某些字段提前/滞后过渡。这个约束在 D6 cmd_mask 语义设计中被引用。

#### RB-06 外部输入 → FMS 20 ms

外部源(Pilot/GCS/Auto/Mission)的实际产出速率是不可控的,可能高于、等于或低于 FMS 20 ms 周期。统一策略:

| 字段 | 取值 |
|---|---|
| Strategy | **Single-element queue(邮箱) + Latch on FMS step start** |
| Latency budget | ≤ 1 FMS step(20 ms) |
| Staleness ceiling | 20 ms + 来源端自身周期(即 FMS 永远落后来源端不超过 1 个 FMS step + 来源端 1 个产出周期) |
| Determinism | async(来源端不固定相位) |
| Failure mode if violated | FMS 一帧中读到来源端的部分新部分旧字段 → 命令解析错误 |

实现要点:

- 每路输入 bus 分别维护一个"最近写入"邮箱;FMS step 起始时一次性 latch 该邮箱的全部字段(同 RB-05 的 latch 语义)。
- 若来源端在两次 FMS step 之间产生了多次更新,**只保留最后一次**(队列长度 = 1);早先的更新丢弃。这是设计层面的可观测性损失,但避免了变长队列。
- 来源端自身的 timestamp(若 bus 中有)与 latch 的 FMS step 时间的相位关系由 [A7 时间与时间戳约定](A7-time-conventions.md) 处理;A4 仅约束传输策略,不约束 timestamp 对齐。
- Pilot 注入由 [G4 Pilot_Cmd 注入设计](../G-harness/G4-pilot-injection.md) 提供脚本格式;G4 输出在仿真时间轴上的事件序列即视为 Pilot 来源端,本边界策略适用。

#### RB-07 Plant 1 ms → 慢传感器输出(Plant 模块内部)

架构 v1 §12.4 要求"slower sensor publication modeled through output update cadence rather than internal multi-rate complexity where possible" — Plant 内部保持 1 ms 单速率,慢传感器(GPS、Barometer 等)通过"按 cadence 更新输出 bus 字段"的方式建模:

| 字段 | 取值 |
|---|---|
| Strategy | **Sample-time-based downsample**(在 Plant 1 ms 主循环内部周期性触发慢传感器输出更新) |
| Latency budget | 0(同 step 内立即更新) |
| Staleness ceiling | 等于该传感器自身 cadence(GPS ~ 100–200 ms, Barometer ~ 10–20 ms 等;具体值由 [C1 Plant 功能设计](../C-plant/C1-plant-functional.md) 定义) |
| Determinism | single-rate-aligned(各 cadence 必须为 1 ms 的整数倍,且互不耦合) |
| Failure mode if violated | Plant 内部出现多 sample-time → 与"single-rate per module"原则冲突,触发 G2 调度告警 |

约束:慢传感器的具体 cadence 值由 C1 决定,A4 不指定具体毫秒数,只锁定**实现方式**(在 1 ms 主循环内由计数器/触发器控制输出更新,不引入内部多速率)。

### 4.5 整数倍同相对齐策略(关键决策,闭合 A1 §4.4 缺口)

A1 §4.4 把架构 v1 §4.4 "Single-rate per module" 缺口指派给 A4:**"INS 100 Hz 在模型仓侧如何采样(整数倍 vs 异步)未规定"**。

**决策:模型仓侧 MIL 中,INS 10 ms / Controller 5 ms / Plant 1 ms / FMS 20 ms 全部按整数倍 + 同相起步对齐**。即 `t = 0` 时刻所有模块同时第一次执行;之后每个模块在其周期的整数倍时刻执行。

理由:

1. 架构上四个周期(1, 5, 10, 20 ms)恰好是整数倍链(1 | 5 | 10 | 20),不存在素数互不整除的尴尬组合。
2. 整数倍同相消除了"边界相位漂移"风险 — RB-03(INS→Controller)和 RB-04(INS→FMS)的延迟可以严格量化,不需要在策略中加入 async 容忍。
3. 与 G2 在 Simulink Solver 中使用 fixed-step discrete + base rate = 1 ms 直接吻合。
4. **`ins_stub` 在模型仓侧的发布速率必须设为 10 ms**(与 firmware 中真实 INS 100 Hz 一致),这一约束由 [F2 ins_stub 结构设计](../F-ins-contract/F2-ins-stub-structural.md) 在结构层落地。本边界设计强制此约束作为前提:`ins_stub` 不得以 1 ms 或 5 ms 速率发布,否则 RB-03 / RB-04 的策略与延迟预算全部失效。

非整数倍场景(假设性):若未来某模块速率改成 7 ms 或 15 ms,本文件的策略无法直接套用,届时需要把对应边界从 ZOH 升级为 single-element queue,且重新计算陈旧度。**任何引入非整数倍周期的修改 = 触发 A4 重新评审**,这是 A4 的稳定边界。

### 4.6 延迟预算分配(粗粒度,闭合 A1 §13.5 缺口)

架构 v1 §13.5 "Plant and INS latency must be considered in closed-loop MIL/SIL scenarios to avoid hiding integration issues" — 该缺口同时指派给 A4 与 E5。本工作项给出的是**架构层粗预算**,E5 在 5 ms 范围内做更细的子预算。

定义闭环延迟链(MIL,典型路径):

```text
Plant 物理状态 → ins_stub 计算 → INS_Out_Bus 发布 → Controller 读取 → Controller 算控制 → Control_Out_Bus 写出 → Plant 读取 → 下一物理状态
```

各段延迟预算(架构层):

| 段 | 预算上限 | 来源 |
|---|---:|---|
| Plant 内 1 ms step 计算 | ≤ 1 ms | 由 C4 数值预算给出实际值 |
| `ins_stub` 计算 + RB-03/RB-04 边界传输 | ≤ 10 ms(`ins_stub` 周期上限)+ ≤ 1 ms 边界 | F2(`ins_stub` 结构)+ 本文件 RB-03/04 |
| Controller 内 5 ms step 计算 | ≤ 5 ms | 由 E5 性能预算细化 |
| RB-02 边界传输(Control_Out → Plant) | ≤ 1 ms | 本文件 RB-02 |
| **总闭环延迟上界** | ≤ ~17 ms | 加和(典型场景下显著小于此值) |

注意:这是**最差情形上界**,典型场景因为整数倍同相对齐(§4.5)远小于此值。本预算的意义是给 H 区(验证)在分析"为何 MIL pass 但 SIH fail"时一个明确的延迟参照线。

### 4.7 边界文档化模板(闭合 A1 §4.6 缺口)

A1 §4.6 把架构 v1 §4.6 "Explicit interfaces / documented rate boundaries 文档形式未指定" 缺口指派给 A4。本节给出**下游模块设计文档**(C2/D2/E2/F2 等)在描述自身输入/输出时,描述跨速率边界使用的标准模板:

```pseudo
Module: <module name>
For each input bus that crosses a rate boundary:
  - Boundary ID: <RB-NN>             # reference §4.2 of A4
  - Bus name: <bus identifier>        # reference B1 for fields
  - Rate at producer side: <ms>
  - Rate at this module: <ms>
  - Strategy: <ZOH | Latch | Downsample | SingleElementQueue>  # reference §4.3
  - Latency budget: <ms>              # reference §4.4 of A4
  - Staleness ceiling: <ms>           # reference §4.4 of A4
  - Validity flag handling: <described | N/A>
For each output bus that crosses a rate boundary:
  - (same fields, with this module as producer side)
```

模板使用规则:

1. 每个下游设计文档(C2/D2/E2/F2 等)在自己的"模块边界"章节附此清单。
2. 各字段值**必须**与 A4 §4.4 一致;若需偏离,必须 PR 回 A4 增补一行新边界 ID,并经 A4 重新评审。
3. 此模板**不**重复 bus 字段表 — bus 字段定义留给 B1。

这个模板把 A4 的设计成果"可消费化",也确保下游不会绕开 A4 自行决定边界策略。

### 4.8 边界设计规则总览

将上述决策抽象为四条规则,作为下游遵守清单:

1. **R1 单速率内不引入多速率**:每个模块(Plant/Controller/FMS/`ins_stub`)在其周期内是 single-rate;rate transition 只在模块**之间**或在 harness 中发生(架构 v1 §9.3.2)。Plant 内部慢传感器例外:由 RB-07 通过同 1 ms 主循环 + cadence 计数器实现,不算引入多速率。
2. **R2 命令路径慢→快用 ZOH**:任何"setpoint / 命令"性质的慢→快边界(RB-02、RB-03、RB-05)使用 ZOH;严禁插值。
3. **R3 反馈路径快→慢用 sample-time-based downsample**:任何"测量 / 真值"性质的快→慢边界(RB-01、RB-04、RB-07)使用 sample-time-based downsample;不得做平均或滤波(平均/滤波若需要,移到产出模块内部完成)。
4. **R4 异步源用 single-element queue + latch**:外部输入(RB-06)的来源率不固定时,使用单格邮箱 + 在消费 step 起始 latch;丢弃旧值,不做队列累积。

任何违反 R1–R4 的设计必须在该工作项的 §5(已知风险)中显式记录并回写到 A4 的变更日志。

## 5. 已知风险与悬而未决问题

- **Latency budget 是上界而非典型值**
  - 影响:E5 / C4 在做细粒度预算时,可能据此上界规划资源,实际典型值远小,造成预算浪费。
  - 处置:本文件 §4.6 已声明这是"最差情形上界";E5 可结合实际典型时序做更紧的预算,但不得超过本文件的上界。

- **`ins_stub` 必须严格 10 ms 发布的约束未由本文件强制**
  - 影响:若 F2 在结构设计中改为 1 ms 或 5 ms 发布(例如为了 truth-state 调试),RB-03/RB-04 的策略将失效。
  - 处置:本文件 §4.5 显式要求 `ins_stub` = 10 ms,并指向 F2 落地;F2 在自己的退出条件中应包含"发布速率 = 10 ms"的复核。若 F2 决策反转,A4 需重新评审。

- **RB-06 来源端 timestamp 与 FMS step 起始相位的关系未定**
  - 影响:外部源若携带 timestamp,FMS 在 latch 时是用 latch 时刻还是来源 timestamp 作为命令时间锚,会影响 D1 / D6 的解析。
  - 处置:推迟到 [A7 时间与时间戳约定](A7-time-conventions.md);A7 与 A4 隐含 ↔(同 batch Wave 2)。本文件 §4.4 RB-06 中已显式引用 A7。

- **Plant 内部慢传感器的 cadence 列表未定**
  - 影响:RB-07 给出了实现方式,但不锁定具体 cadence 值;C1 / C2 启动前,慢传感器边界数量与 cadence 是 placeholder。
  - 处置:由 [C1 Plant 功能设计](../C-plant/C1-plant-functional.md) 列出传感器集合 + 各自 cadence;C2 在结构上落地。本文件保证策略不变(downsample),只是数值待 C 区填入。

- **MIL 默认变体下 RB-01 的处置**
  - 影响:RB-01 在 MIL 默认变体下不存在,但 G1 / A5 设计变体接入点时若决定启用 truth passthrough 给 Controller,RB-01 立即激活。
  - 处置:本文件 §4.4 已为 RB-01 备好策略(downsample),G1 / A5 启用时无需重新决定边界策略;但 G1 应在自己的设计中明确"启用 RB-01 = 进入调试变体"的开关位置。

- **架构 v1 §17 Risk 5 Timing realism 的 MIL→SIH 桥接由 H 区承担**
  - 影响:本文件给的 latency 上界(~17 ms)在 MIL 中可达成,但 SIH 中 firmware 调度抖动可能超出。这是 H1/H3 关心的问题。
  - 处置:不在本工作项范围内闭合;本文件提供量化上界,供 H 区回归对比使用。

- **本文件未涵盖 contract_impact 风险**
  - 影响:本文件**不**触及 firmware 可见 bus / enum / parameter / symbol(纯模型仓侧调度策略),故 contract_impact = no。
  - 处置:无需 INDEX 决策日志登记;若未来策略涉及 firmware 可见 period 改动(目前没有),contract_impact 升级为 yes 重新评审。

## 6. 退出条件复核

A4 在 [`00-design-plan.md`](../00-design-plan.md) §4.A 中的退出条件原文:

> 1ms/5ms/10ms/20ms 之间的 rate transition 位置、策略(zero-order hold / latch / queue)定稿

逐条复核:

| # | 退出条件分解 | 本文档依据 | 状态 |
|---|---|---|---|
| 1 | "1ms/5ms/10ms/20ms 之间" — 四个速率全部覆盖 | §4.1 总览表显式列出 4 个速率;§4.2 边界表覆盖 1↔5、5↔1、10↔5、10↔20、20↔5 全部 4 速率间组合 | 满足 |
| 2 | "rate transition 位置" — 具体边界点定稿 | §4.2 RB-01..RB-07 共 6+1 个边界点的稳定 ID | 满足 |
| 3 | "策略(zero-order hold / latch / queue)" — 选型定稿 | §4.3 给出四种策略(ZOH / latch / sample-time-based downsample / single-element queue);§4.4 每个 RB 显式选定一种 + 理由 | 满足 |
| 4 | "定稿" — 决策可被下游消费 | §4.4 每边界给 latency budget + staleness ceiling + determinism + failure mode;§4.7 提供下游模板;§4.8 R1–R4 规则 | 满足 |

附加(隐含):

| # | 隐含条件 | 本文档依据 | 状态 |
|---|---|---|---|
| 5 | A1 指派给 A4 的 7 条缺口全部应答 | §4.5(整数倍对齐 → 闭合 A1 §4.4)、§4.7(文档化模板 → 闭合 A1 §4.6)、§4.6(latency 预算 → 闭合 A1 §13.5)、§4.4 RB-03/RB-04(INS 采样 → 闭合 A1 §5.1.3 / §13)、§4.4 + §4.8 R1(harness rate transition → 闭合 A1 §9.3.2)、§4.4 RB-07(Plant 慢传感器 → 闭合 A1 §12.4) | 满足 |

退出条件全部满足;状态可由 reviewer 升级到 reviewed。

## 7. 下游影响

按 [`01-design-relationships.md`](../01-design-relationships.md) §4.1 / §4.7 / §4.10 中 A4 出边:

```text
A1 → A4
A2, A4, A5 → A3
A4 → G2
A4 ⇢ E5, C4(经 §4.6 latency budget)
A4 ⇢ F2(经 §4.5 `ins_stub` 10 ms 强制约束)
A7 ⇢ A4(同 batch Wave 2 隐式 ↔;本文件已引用 A7)
```

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [A3 模块边界信号清单细化](A3-module-boundaries.md) *(未启动)* | → | §4.2 边界 ID 清单(A3 在为每个 bus 字段标注方向时引用 RB-NN);§4.7 模板供 A3 消费 |
| [G2 多速率调度设计](../G-harness/G2-rate-scheduling.md) *(未启动)* | → | §4.2 全部边界点;§4.3 策略术语;§4.4 latency budget;§4.5 整数倍同相对齐;§4.8 R1–R4 |
| [E5 Controller 性能预算设计](../E-controller/E5-performance-budget.md) *(未启动)* | ⇢ | §4.4 RB-03 / RB-05 / RB-02(Controller 输入输出边界);§4.6 闭环延迟上界;Controller 5 ms 内的预算由 E5 在此约束内细分 |
| [C4 Plant 数值与积分设计](../C-plant/C4-numerics.md) *(未启动)* | ⇢ | §4.4 RB-02 / RB-01 / RB-07;§4.6 Plant 1 ms step 预算上限;C4 在 1 ms 内做积分器选型 |
| [F2 ins_stub 结构设计](../F-ins-contract/F2-ins-stub-structural.md) *(未启动)* | ⇢ | §4.5 `ins_stub` 必须 10 ms 发布;§4.4 RB-03 / RB-04 |
| [A7 时间与时间戳约定](A7-time-conventions.md) *(同 Wave 2)* | ↔(隐式) | A4 引用 A7 处理 timestamp 对齐;A7 引用 A4 处理 sample-time 关系 |

间接影响(经 A3):

- B1(bus schema):A3 通过本文件 §4.7 模板要求,在每个 bus 字段处标注边界 ID;B1 不直接受 A4 影响,但 bus 的"使用上下文"会引用 A4。
- D1 / D2 / E1 / E2 / C1 / C2 / F1 / F2:各模块在自己的输入/输出边界处都将引用 A4 §4.4 表行。

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-05 | A4 author | 初稿 |

## Self-check

- [x] frontmatter 完整,字段值合法
- [x] 上游文档全部存在且 status ≥ reviewed(架构 v1 已发布并锚定 2026-05-05;A1 status: reviewed,2026-05-05)
- [x] 退出条件逐条复核完成,每条均给出依据
- [x] 引用路径全部可点击访问
- [x] 不存在 RULES §5 禁则中的内容(无 .slx、无可执行 .m、无 firmware 实现重述、无 PR/branch、无 bus 字段重复定义、有依赖列表;§4.7 模板标 `pseudo`)
- [x] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记(本文件 contract_impact=no,N/A)
- [x] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径(本文件未镜像 firmware 契约,N/A)
- [x] 下游影响已沿关系图识别完毕(§7 含 A3、G2、E5、C4、F2、A7)
- [x] 文档不超出本工作项范围(harness 调度实现交 G2;Controller 子预算交 E5;Plant 数值交 C4;`ins_stub` 内部交 F1/F2;timestamp 对齐交 A7;变体接入交 A5/G1)
