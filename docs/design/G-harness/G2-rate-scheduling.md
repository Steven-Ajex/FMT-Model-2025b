---
work_item: G2
title: 多速率调度设计
upstream: ["架构 v1", "A4", "A7"]
contract_impact: no
status: draft
authored_at: 2026-05-10
last_reviewed_at:
reviewer_verdict: none
---

# G2 多速率调度设计

## 1. 目的

为 MIL 顶层 harness 在 Simulink 中**整体落地多速率调度**:固定 Solver 配置、四个模块(Plant / Controller / `ins_stub` / FMS)的 Sample Time 锁定、A4 §4.2 七条 rate-transition 边界(RB-01..RB-07)在 harness 中的块级位置与参数、Sample Time 颜色检查清单、SingleTasking vs MultiTasking 抉择、codegen 兼容性约束(架构 v1 §4.5),并给出 20 ms 主调度窗口的时间线(schedule 图)。**A4 决策的策略 / 延迟预算 / 陈旧度上界由 G2 echo,不重定义**。

## 2. 范围

**在范围:**

- Solver pane 配置(type / Solver / Fixed-step size / Tasking mode)
- 四个模块的 Sample Time 锁定(Plant 0.001 / Controller 0.005 / `ins_stub` 0.010 / FMS 0.020 s)
- A4 §4.2 七条 rate-transition 边界在 harness 中的块部署位置与参数
- Sample Time **颜色检查清单**(Display→Sample Time→Colors 后预期颜色)
- Tasking mode 抉择:Phase 2 选 SingleTasking,MultiTasking 风险 forward-cite E5
- Codegen 兼容性硬约束(架构 v1 §4.5)在 Solver 层的体现
- 20 ms 主调度窗口的时间线 ASCII 图

**不在范围(由其他工作项处理):**

- Rate-transition **策略选型与延迟预算**(ZOH / latch / downsample / single-element queue)— 由 [A4 跨速率边界设计](../A-architecture/A4-rate-boundaries.md) §4.3 / §4.4 拥有;G2 仅 echo
- 各模块周期数值本身(1/5/10/20 ms)— 由架构 v1 §13 与 A4 §4.1 拥有
- Harness 顶层子系统拓扑、引用模型策略、变体接入点 — 由 G1(Wave 10 同 batch)拥有
- logsout 信号采样时戳元数据 — 由 G3(Wave 10 同 batch)拥有
- Pilot_Cmd 注入脚本格式 — 由 G4 拥有
- Controller 5 ms 内的子预算分配(各环路 µs 级) — 由 [E5 Controller 性能预算设计](../E-controller/E5-performance-budget.md) 拥有(Wave 10 同 batch)
- Plant 1 ms 内积分器选择与数值稳定性 — 由 [C4 Plant 数值与积分设计](../C-plant/C4-numerics.md) 拥有
- `ins_stub` 内部如何派生 `INS_Out_Bus` 字段 — 由 F1/F2 拥有
- timestamp 单位 / epoch / 回卷 / dt 来源 — 由 [A7 时间与时间戳约定](../A-architecture/A7-time-conventions.md) 拥有
- ert.tlc 完整配置项清单(ParameterPooling / RTWInlineParameters 等)— 由 I4 拥有

## 3. 依赖

| 上游 | 引用位置 | 用途 |
|---|---|---|
| 架构 v1 §4.5 Deterministic codegen | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | "ert.tlc, fixed-step discrete, no dynamic memory, no recursion, no variable-size signals" — Solver 层硬约束 |
| 架构 v1 §13 Timing constraints | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | Plant 1ms / Controller 5ms / FMS 20ms / INS 10ms 周期表(G2 echo) |
| 架构 v1 §9.3.2 | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | "Cross-rate changes occur only in harness or scheduler wiring" — 把 rate transition 限定在 harness 层 |
| [A4 跨速率边界设计](../A-architecture/A4-rate-boundaries.md) (status: reviewed) | §4.1, §4.2, §4.3, §4.4, §4.5, §4.8 | 模块周期表、RB-01..RB-07 边界 ID、四种策略术语、各边界 latency budget / staleness ceiling、整数倍同相对齐决策、R1–R4 规则(全部 echo,不重定义) |
| [A7 时间与时间戳约定](../A-architecture/A7-time-conventions.md) (status: reviewed) | §4.5 dt 来源 = 模块固定步长 | Solver 固定步长决策与 timestamp 用途分离 |
| [C4 Plant 数值与积分设计](../C-plant/C4-numerics.md) (status: reviewed) | §4.7 | RK4 fixed-step 1ms + 6 codegen guards,Solver 层"无连续状态、纯离散"承诺一致 |
| [F2 ins_stub 结构设计](../F-ins-contract/F2-ins-stub-structural.md) (status: reviewed) | — | `ins_stub` 在模型仓侧的 sample time = 0.010 s(A4 §4.5 强制约束的落地点) |

约定:本文件在涉及 rate-transition 策略 / 延迟预算时**只引用 A4 §4.4 表行**,不复述具体 latency / staleness 数值。本文件在涉及 timestamp / dt 时**只引用 A7 §4.5 决策**,不重述 epoch / 回卷。

## 4. 设计内容

### 4.1 Solver pane 配置

模型(harness `.slx`)Configuration Parameters → Solver pane 的锁定配置:

| 项 | 值 | 依据 |
|---|---|---|
| Type | **Fixed-step** | 架构 v1 §4.5 "fixed-step discrete";A4 §4.5 整数倍同相对齐;A7 §4.5 dt = 模块固定步长 |
| Solver | **discrete (no continuous states)** | 架构 v1 §4.5 + C4 §4.7 — 模型整体无连续状态(Plant 内部 RK4 由 C4 实现为离散积分器,见下补充) |
| Fixed-step size (fundamental sample time) | **0.001** s(1 ms) | 是 1 / 5 / 10 / 20 ms 的最大公约数(GCD),即基础 tick |
| Periodic sample time constraint | Unconstrained | 各模块 sample time 在 §4.2 显式声明 |
| Tasking mode (Solver pane → Tasking and sample time options) | **SingleTasking**(Phase 2;§4.5 详述) | 简化 codegen、消除任务优先级反转风险;MultiTasking 推迟到 Phase 3+ 评估 |
| Automatically handle rate transition for data transfer | **Off**(显式 Rate Transition 块) | A4 §4.8 R1 要求 rate transition 仅出现在 harness 层并被显式表达 |
| Higher priority value indicates higher task priority | N/A(SingleTasking 模式下不生效) | Phase 2 |
| Allow tasks to execute concurrently on target | Off | 单核确定性 |

补充说明:

- **"Solver = discrete"** 与 C4 §4.7 不矛盾:C4 选择 RK4 fixed-step 1 ms 作为 Plant **内部**积分方法,但实现时使用离散块(Discrete-time Integrator 配 Integration method = "Integration RK4",或自定义离散 RK4 状态展开)使整个模型对外呈现"无连续状态"。这是架构 v1 §4.5 + 架构 v1 §13 单速率原则的必然结果。
- **Fixed-step size = 1 ms** 是 base step。Simulink solver 据此推进;1 ms 是 Plant 周期,且为 5/10/20 ms 的整数除子,所有更慢模块 sample time 都是 1 ms 的整数倍 → Solver 不会出现非整数 tick。
- **Stop time** 由 G1 / I5 在仿真运行时设置(如 60 s 验证场景),不在 Solver pane 静态锁定。

### 4.2 各模块 Sample Time 锁定

承袭架构 v1 §13 + A4 §4.1,harness 中四个模块子系统的 Sample Time 设置:

| 模块(harness 子系统) | Sample Time(秒) | Rate(Hz) | 数据来源 | 拥有方 |
|---|---:|---:|---|---|
| Plant | **0.001** | 1000 | 架构 v1 §13;A4 §4.1 | 模型仓 |
| Controller | **0.005** | 200 | 架构 v1 §13;A4 §4.1 | 模型仓 |
| `ins_stub` | **0.010** | 100 | A4 §4.5 强制(对齐 firmware INS 100 Hz);F2 落地 | 模型仓(消费 firmware 拥有的 `INS_Out_Bus` 契约) |
| FMS | **0.020** | 50 | 架构 v1 §13;A4 §4.1 | 模型仓 |

实现要点:

- 每个模块子系统的 sample time 在子系统块的"Sample time" 参数显式设置(**不**用 `-1` inherited)。这一硬约束对应 A4 §4.8 R1 "single-rate inside each module"。
- 模块**内部**所有块的 sample time 一律 inherited(`-1`),由子系统外壳的 sample time 一次性传播。这避免内部出现多速率并触发 G2 调度告警。
- `ins_stub` sample time = 0.010 s 是 [A4 §4.5 决策](../A-architecture/A4-rate-boundaries.md) 与 [F2 结构设计](../F-ins-contract/F2-ins-stub-structural.md) 的硬约束:**G2 不允许任何 harness 变体改写 `ins_stub` 周期**。如调试需要 1 ms truth state passthrough,使用 RB-01 路径(§4.3),不动 `ins_stub` 周期。
- t = 0 时刻所有四个模块同相起步(integer-multiple aligned phase),由 A4 §4.5 "整数倍同相对齐"强制 — Simulink fixed-step discrete solver 默认即如此(所有 Ts 同时从 t=0 计数),G2 不需要额外配置。

### 4.3 Rate transition 块在 harness 中的部署(echo A4 §4.4)

下表把 A4 §4.2 / §4.4 的七条边界落地到 harness 中具体 Rate Transition 块的位置与必填参数。**策略列、Latency budget 列、Staleness ceiling 列由 A4 §4.4 拥有,本表 echo 不重定义**;G2 只新增"块级参数"列。

| RB ID | 来源 → 去向(Hz) | 速率方向 | A4 策略(echo) | Rate Transition 块部署位置 | 块级参数(Simulink Rate Transition) |
|---|---|---|---|---|---|
| RB-01 | Plant 1000 → Controller 200 *(仅 harness 调试变体)* | 快→慢 | Sample-time-based downsample | Plant 子系统输出 → Controller 子系统输入(仅 truth-passthrough variant 启用) | Output port sample time = `0.005` (explicit); Initial condition = ZERO; Ensure data integrity = on; Ensure deterministic data transfer = on |
| RB-02 | Controller 200 → Plant 1000 | 慢→快 | ZOH | Controller 子系统输出 → Plant 子系统输入 | Output port sample time = `0.001` (explicit); Initial condition = ZERO(per A6 §4.4 actuator reset class); Ensure data integrity = on; Ensure deterministic data transfer = on |
| RB-03 | `ins_stub` 100 → Controller 200 | 慢→快 | ZOH | `ins_stub` 输出 `INS_Out_Bus` → Controller 输入 | Output port sample time = `0.005` (explicit); Initial condition = ZERO(INS validity = 0,per A6 §4.7.1 INS gate);Ensure data integrity = on; Ensure deterministic data transfer = on |
| RB-04 | `ins_stub` 100 → FMS 50 | 快→慢 | Sample-time-based downsample | `ins_stub` 输出 `INS_Out_Bus` → FMS 输入(分支自 RB-03 共享 `INS_Out_Bus` 信号) | Output port sample time = `0.020` (explicit); Initial condition = ZERO; Ensure data integrity = on; Ensure deterministic data transfer = on |
| RB-05 | FMS 50 → Controller 200 | 慢→快 | ZOH(主字段)+ Latch(同帧一致性) | FMS 子系统输出 `FMS_Out_Bus` → Controller 子系统输入 | Output port sample time = `0.005` (explicit); Initial condition = HARDCODED(per A6 §4.4 default cmd_mask = 0,mode = DISARMED);Ensure data integrity = on; Ensure deterministic data transfer = on |
| RB-06 | 外部输入(Pilot/GCS/Auto/Mission)→ FMS 50 | async / 来源率不固定 | Single-element queue + Latch on FMS step start | Harness 外部输入子系统(由 G4 提供 Pilot_Cmd 来源)→ FMS 子系统输入 | Output port sample time = `0.020` (explicit); Initial condition = ZERO; Ensure data integrity = on; Ensure deterministic data transfer = on; **来源端时间戳处理 deferred to A7 §4.5** |
| RB-07 | Plant 1000 内部 → 慢传感器输出 cadence | 1ms → cadence | Sample-time-based downsample(Plant 主循环内由 cadence 计数器触发) | Plant 子系统**内部**(harness 不可见)— G2 仅校验 Plant 子系统对外仍为单 0.001 sample time | N/A(由 C2 在 Plant 内部以 enabled-output 或 trigger pattern 实现;harness 中不出现 Rate Transition 块) |

通用块参数说明:

- **Output port sample time inheritance:** 所有 Rate Transition 块**禁止**使用 `-1`(inherited)。必须显式写明目标 sample time(秒数),理由是架构 v1 §4.5 deterministic codegen + A4 §4.8 R1 要求 rate transition 显式可见。
- **Initial condition source(ZERO vs HARDCODED):** 与 [A6 Init/Reset 状态契约](../A-architecture/A6-init-reset-contract.md) §4.4 的复位类别(PARAM / HARDCODED / ZERO / PRESERVED)一致;G2 默认选 ZERO,RB-05 因 `cmd_mask` / `mode` 默认值非零(disarmed 而非 0 raw)选 HARDCODED。具体值由 A6 / B3 owner,本表只标分类。
- **Ensure data integrity / Ensure deterministic data transfer:** 全部 on。"Data integrity = on" 防止 bus 拆字段过渡(对应 A4 §4.4 RB-05 latch 子语义);"deterministic = on" 把任务优先级影响清零(SingleTasking 下默认满足,但显式打开为 MultiTasking 升级做前置)。
- **harness 不允许出现"自动"插入的 rate transition:** §4.1 已关闭 "Automatically handle rate transition for data transfer";若 Solver 报错 "rate transition required between …", 必须由作者**显式**插入块,这是 A4 §4.8 R1 + 架构 v1 §9.3.2 落地。

### 4.4 Sample Time 颜色检查清单

Simulink "Display → Sample Time → Colors" 视图把每个块按 sample time 着色。本节给出**预期颜色矩阵**作为 harness 集成完成后的可视化验收清单。

#### 4.4.1 颜色映射(Simulink 默认调色板)

| Sample Time(秒) | 周期(ms) | 频率(Hz) | 预期颜色 | 备注 |
|---|---:|---:|---|---|
| 0.001 | 1 | 1000 | **Red** | Plant 子系统(整体)+ 其内部所有块 |
| 0.005 | 5 | 200 | **Green** | Controller 子系统(整体)+ 其内部所有块;Simulink 默认 5 ms 取色为绿 |
| 0.010 | 10 | 100 | **Cyan**(浅蓝) | `ins_stub` 子系统 + `INS_Out_Bus` 信号线在 Rate Transition 上游段 |
| 0.020 | 20 | 50 | **Yellow** | FMS 子系统 + `FMS_Out_Bus` 信号线在 Rate Transition 上游段 |
| Multi-rate boundary | — | — | **Magenta** / **Hybrid 灰** | Rate Transition 块本身被 Simulink 标识为 "hybrid" 或 mixed-rate 颜色,提示边界 |

> 注:Simulink 的具体颜色编码(red/green/cyan/yellow/magenta)源自 MathWorks 文档默认配色表;若用户工作站使用自定义 colormap,验收人需以自身配色表为准但保持"4 个不同纯色 + 1 个 hybrid 标识色"的语义一致性。本文件不锁定 RGB 数值,仅锁定**类目**(每个 sample time 必须有唯一颜色,Rate Transition 块必须为 hybrid 标识色)。

#### 4.4.2 颜色检查程序(harness 集成后必跑)

```pseudo
Procedure: Sample Time Color Check
1. Open harness top-level model.
2. Menu: Display → Sample Time → Colors (toggle ON).
3. Verify each module subsystem outline color matches §4.4.1 row.
   - Plant subsystem outline   = Red
   - Controller subsystem outline = Green
   - ins_stub subsystem outline = Cyan
   - FMS subsystem outline      = Yellow
4. Verify each Rate Transition block from §4.3 table appears with hybrid/multi-rate indicator color.
5. Verify NO "black"(Inherited unspecified)blocks exist. Black indicates a block did not resolve a sample time → schedule violation.
6. Verify NO unexpected color appears
   (e.g., a 0.002 s block would show as a non-mapped color → indicates an unintended rate).
7. Within each module subsystem, drill down once and verify all internal blocks
   inherit the parent color (per §4.2 implementation rule:
   "module-internal blocks use Ts = -1 inherited").
8. Record check result in harness build report (G3 §<TBD> manifest).
```

#### 4.4.3 失败处置

- 若步骤 5 出现黑色块 → 该块未解析到任何 sample time。fix:在该块上层显式设置 sample time(子系统外壳)或检查该块是否实际未连接。
- 若步骤 6 出现非映射颜色 → harness 中混入了未声明的 sample time(常见错因:从外部库引入的块带自身 Ts)。fix:回 §4.2 表锁定四个 sample time;非这四者的块不允许出现。
- 若步骤 7 出现子系统内部颜色与外壳不一致 → 模块内部出现了多速率,违反 A4 §4.8 R1。fix:把不一致块的 Ts 改回 inherited。

### 4.5 Tasking mode 抉择与 MultiTasking 风险(forward-cite E5)

#### 4.5.1 Phase 2 决策:**SingleTasking**

依据:

1. **codegen 可预测性**:SingleTasking 下生成的 ert.tlc 代码以单 step 函数顺序执行所有任务,无任务调度框架介入,与 firmware 现有 `task_vehicle.c` 顺序调用 `Plant/FMS/Controller_step` 的运行模式一致。
2. **优先级反转风险归零**:Phase 2 没有任务抢占,不可能出现 INS 慢任务被 Plant 快任务挤占的反转。
3. **调试可观测性**:SingleTasking 下 logsout 时戳与 step 计数严格对应,G3 设计的"采样时戳规则"实施成本低。
4. **A4 整数倍同相对齐前提满足**:SingleTasking 下所有 sample time 在每个 fundamental tick(1 ms)上同相执行 — 与 A4 §4.5 决策天然一致。

#### 4.5.2 MultiTasking 假设性升级路径(forward-cite E5)

如果 Phase 3+ 因为 Controller 5 ms 实测预算逼近上界(由 [E5 Controller 性能预算设计](../E-controller/E5-performance-budget.md) 定量给出)需要给慢任务让路时,可考虑切换 MultiTasking。届时**任务优先级建议**:

| Sample time | Simulink Priority(数值越小优先级越高) | 模块 |
|---|---:|---|
| 0.001 s | **0**(最高) | Plant |
| 0.005 s | **1** | Controller |
| 0.010 s | **2** | `ins_stub` |
| 0.020 s | **3**(最低) | FMS |

风险:

- **优先级反转**:`ins_stub` 在 10 ms 边界产出 `INS_Out_Bus`,若被更高优先级 Plant/Controller 持续抢占,可能导致 INS 数据陈旧度超过 A4 §4.4 RB-03 / RB-04 所规定的 10 / 20 ms 上界。**检测手段**:I3 契约 diff 之外的 post-codegen profiling(由 E5 设计阶段定义采样方法),或 G3 logsout 中的 `INS_Out_Bus.timestamp` vs harness wall-time 对比。
- **A6 reset 协议复杂化**:MultiTasking 下 `*_init` 调用顺序不严格保证;A6 §4.2 单调启动顺序(Plant_init → ins_stub_init → Controller_init → FMS_init)需在 MultiTasking 入口手动序列化。
- **A4 §4.5 整数倍同相对齐削弱**:MultiTasking 容许相位漂移;若启用,RB-03 / RB-04 的 staleness ceiling 需要 +1 个最高优先级 step 的容忍量(±1 ms),A4 需重新评审。

> Phase 2 不启用 MultiTasking;以上仅作"deferred concerns logged"。E5 在 Wave 10 完稿时若需触发,通过 INDEX 决策日志登记。

### 4.6 Codegen 兼容性硬约束(echo 架构 v1 §4.5)

Solver 层与多速率调度层必须满足架构 v1 §4.5 的全部 codegen 约束。本节 echo,**不重定义**:

| 约束 | 落地点(本文件) | 关联工作项 |
|---|---|---|
| **不允许变步长 Solver** | §4.1 Type = Fixed-step | 架构 v1 §4.5 |
| **不允许动态内存** | Rate Transition 块禁用变长队列(§4.3 单格);ins_stub / FMS / Controller 子系统禁用变长信号 | 架构 v1 §4.5;A4 §4.3(禁多元素 FIFO) |
| **不允许递归** | 模块内部不出现递归子系统;Rate Transition 块本身无递归 | 架构 v1 §4.5;I4 codegen 配置脚本统一禁用 |
| **不允许可变尺寸信号** | Bus 字段 schema 由 B1 锁定;Rate Transition 块不变维 | 架构 v1 §4.5;B1 |
| **离散 fixed-step 1 ms base tick** | §4.1 Fixed-step size = 0.001 | 架构 v1 §4.5;A4 §4.5 |
| **所有状态必须有定义的初始条件** | §4.3 表 "Initial condition" 列;模块内状态 init 由 A6 owner | A6 §4.4 |

任何违反上述六项的 harness 修改 = 触发 G2 重新评审。

### 4.7 调度图(20 ms 主窗口时间线)

20 ms 是四个周期的最小公倍数(LCM(1, 5, 10, 20) = 20),即整个调度模式的重复周期。下图展示一个 20 ms 主窗口内每个模块何时执行(`X` = 该模块在该 1 ms tick 上 step;`.` = 不 step):

```text
Tick (ms):    0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18 19  | 20 (= 0 of next window)
              =====================================================================
Plant   (1ms) X  X  X  X  X  X  X  X  X  X  X  X  X  X  X  X  X  X  X  X  | X
Ctrl    (5ms) X  .  .  .  .  X  .  .  .  .  X  .  .  .  .  X  .  .  .  .  | X
ins_stub(10ms)X  .  .  .  .  .  .  .  .  .  X  .  .  .  .  .  .  .  .  .  | X
FMS    (20ms) X  .  .  .  .  .  .  .  .  .  .  .  .  .  .  .  .  .  .  .  | X
              =====================================================================

Co-execution at t=0  (harness boot):       Plant + Controller + ins_stub + FMS  (all four step together)
Co-execution at t=10:                      Plant + ins_stub                     (RB-03 / RB-04 produce)
Co-execution at t=5/15:                    Plant + Controller                   (RB-02 / RB-03 consume)
Plant-only ticks (15 of 20 in window):     1, 2, 3, 4, 6, 7, 8, 9, 11, 12, 13, 14, 16, 17, 18, 19
```

观察(均 echo A4 §4.5):

1. **t = 0 全模块同相起步**:四个模块在 harness boot 后第一个 tick 同时第一次 step。这是 A4 §4.5 整数倍同相对齐决策的可视化呈现。
2. **t = 10 是 INS 在窗口内的第二次产出**:Plant 与 `ins_stub` 同 tick 执行,RB-03 / RB-04 此 tick 边界数据更新最及时;Controller 在 t = 10 不 step(下次 step 是 t = 15),所以 RB-03 的实际消费滞后约 5 ms(典型,符合 A4 §4.4 RB-03 staleness 5 ms 典型值)。
3. **FMS 在窗口内只 step 1 次(t = 0)**:这是为何 RB-05 需要 latch — FMS 一帧锁存的 `FMS_Out_Bus` 必须在接下来 4 个 Controller step(t = 5, 10, 15)中保持 bus 字段集体一致。
4. **窗口在 t = 20 处重复**:Solver 周期性进入 t = 20 / 40 / 60 …;harness 的整个调度模式以 20 ms 为基础重复。

### 4.8 跨引用:G2 与 G1 / G3 / E5 / A6 / A7 的接口

本节列出 G2 与同 Wave 10 batch 兄弟(G1 / G3 / E5)以及上游(A4 / A6 / A7)的接口承诺,作为**消费方读取本设计的速查表**:

| 邻接工作项 | G2 提供 / 消费 |
|---|---|
| [G1 MIL 顶层结构设计](G1-mil-toplevel.md)(同 Wave 10) | G2 提供 §4.2 sample time 锁定 + §4.3 Rate Transition 部署位置作为 G1 子系统连线规则;G1 提供 harness 顶层子系统数与命名,本文件 §4.4 颜色检查依赖 G1 的子系统拓扑 |
| [G3 日志与可观测性设计](G3-logging.md)(同 Wave 10) | G2 提供 §4.7 调度图作为 G3 logsout 时戳采样规则的输入(慢信号 logsout 的 sample time 必须与产出端一致);G3 提供 manifest 字段供 §4.4.2 步骤 8 写入 |
| [E5 Controller 性能预算设计](../E-controller/E5-performance-budget.md)(同 Wave 10) | G2 提供 §4.1 SingleTasking 决策与 §4.5.2 MultiTasking 升级前置;E5 在 5 ms 内做子预算时不得超出本文件 §4.7 的 Controller step 间距 |
| [A6 Init/Reset 状态契约](../A-architecture/A6-init-reset-contract.md) | G2 §4.3 Initial condition 列引用 A6 §4.4 复位类别(PARAM/HARDCODED/ZERO/PRESERVED) |
| [A7 时间与时间戳约定](../A-architecture/A7-time-conventions.md) | G2 §4.1 fixed-step decision 与 A7 §4.5 dt 来源决策一致(模块内部 dt = 固定步长,timestamp 仅辅助);RB-06 来源端 timestamp 锚定问题完全 deferred 到 A7 §4.5 |
| [A4 跨速率边界设计](../A-architecture/A4-rate-boundaries.md) | G2 全程 echo;§4.3 表是 A4 §4.4 在 harness 中的块级落地;G2 不修改 A4 任何决策 |

## 5. 已知风险与悬而未决问题

- **Sample Time 颜色 RGB 编码不锁定**
  - 影响:不同 MATLAB 版本 / 用户自定义 colormap 下 §4.4.1 颜色名(red/green/cyan/yellow)对应的 RGB 可能漂移;颜色检查清单 §4.4.2 依赖人工确认而非自动化。
  - 处置:本文件 §4.4.1 注脚已声明仅锁定"4 个不同纯色 + 1 个 hybrid 标识色"语义类目;实施阶段若需自动化,移交 I5 sim runner 通过 Simulink API(`get_param(blkH, 'CompiledSampleTime')`)读取实际 sample time 数组对照 §4.2 表,避免依赖颜色比对。

- **MultiTasking 升级路径未在 Phase 2 验证**
  - 影响:§4.5.2 列出的 priority 编号是设计建议,Phase 3+ 若启用未必能即插即用 — INS gate / A6 init 顺序均需重新设计。
  - 处置:推迟到 Phase 3 SIH/SIL;由 E5 触发,届时通过 INDEX 决策日志登记并 G2 重新评审。

- **`ins_stub` sample time 严格 0.010 s 是硬约束,但 G2 不强制执行检测**
  - 影响:harness 作者若误把 `ins_stub` Ts 改为 0.001 / 0.005,§4.4 颜色检查会发现(颜色不在预期映射),但 §4.3 Rate Transition 表的 RB-03 / RB-04 输出 sample time 设置不会自动同步,可能编译通过但语义错误。
  - 处置:G2 §4.4.2 步骤 6 显式列出"未映射颜色 = fail";I4 codegen 配置可加一条 build-time assert(`get_param('harness/ins_stub', 'SampleTime')` == `0.010`);具体 assert 由 I4 owner。

- **t = 0 全模块同相起步在某些 harness 引入模式下可能被破坏**
  - 影响:若用户使用 Initial Conditions 块或 Function-Call Subsystem 触发某模块,可能导致该模块第一次 step 不在 t = 0 而在 t = 0 + δ。这会破坏 A4 §4.5 整数倍同相对齐决策。
  - 处置:G2 §4.2 已要求"模块子系统外壳显式 sample time"(不用 trigger);G1 在顶层拓扑设计中需保证四个模块均为 sample-time-driven 子系统(不是 triggered/enabled)。本文件无法在 G1 启动前完全闭合该风险,标 open,在 G1 reviewer 评审时复核。

- **Solver "discrete (no continuous states)" 与 Plant 内部 RK4 的协同**
  - 影响:C4 §4.7 选择 RK4 fixed-step 1 ms 作为 Plant 物理积分方法;若实现时使用 Simulink 的连续 Integrator 块,Solver 设置就必须改为含连续状态(违反架构 v1 §4.5)。
  - 处置:本文件 §4.1 已声明 Plant 内部 RK4 通过离散 Integrator 块或自定义离散 RK4 状态展开实现,模型对外仍是 "discrete"。该约束由 C4 在结构设计层面承担;G2 不重述实现细节,但 §4.6 codegen 兼容性表锁定"不允许变步长 Solver / 不允许连续状态"作为 invariant。

- **本文件未涵盖 contract_impact 风险**
  - 影响:本文件**不**触及 firmware 可见 bus / enum / parameter / symbol(纯模型仓侧 Solver 与调度配置),故 contract_impact = no。
  - 处置:无需 INDEX 决策日志登记;若未来 SingleTasking → MultiTasking 升级修改了 firmware 可见 `*_EXPORT.period` 数值(目前不会),contract_impact 升级为 yes 重新评审。

## 6. 退出条件复核

G2 在 [`00-design-plan.md`](../00-design-plan.md) §4.G 中的退出条件原文:

> Solver 配置、Sample Time 颜色检查清单、调度图

逐条复核:

| # | 退出条件分解 | 本文档依据 | 状态 |
|---|---|---|---|
| 1 | "Solver 配置" — pane 设置定稿 | §4.1 给出 Solver pane 完整字段表(Type / Solver / Fixed-step size / Tasking mode / Auto rate transition / 优先级配置);§4.6 echo 架构 v1 §4.5 codegen 硬约束 | 满足 |
| 2 | "Sample Time 颜色检查清单" — 可执行的颜色验收程序 | §4.4.1 颜色映射表(4 sample time + 1 hybrid);§4.4.2 8 步检查程序(伪代码);§4.4.3 三类失败处置 | 满足 |
| 3 | "调度图" — 20 ms 主窗口时间线 | §4.7 ASCII 调度图(覆盖 t = 0..20 ms 全部 21 个 tick),含 4 类 co-execution 注解 + 4 条窗口观察 | 满足 |

附加(隐含,由本工作项的上游决策与同 Wave batch 决定):

| # | 隐含条件 | 本文档依据 | 状态 |
|---|---|---|---|
| 4 | A4 §4.2 全部 7 条 RB-NN 边界在 harness 中均有部署位置 | §4.3 表覆盖 RB-01..RB-07(RB-07 标 "Plant 内部,harness 不可见") | 满足 |
| 5 | Tasking mode 抉择(SingleTasking / MultiTasking)有理由 | §4.5.1 Phase 2 选 SingleTasking 给出 4 条依据;§4.5.2 MultiTasking 升级路径 forward-cite E5 | 满足 |
| 6 | 不重定义模块周期或 RB-NN 策略 | §4.2 / §4.3 全部 echo A4 / 架构 v1 §13;策略列标 "(echo)" | 满足 |
| 7 | 与 A7 dt 来源决策一致 | §4.1 fixed-step + §4.6 codegen 硬约束 + §4.8 接口表均显式引用 A7 §4.5 | 满足 |

退出条件全部满足;状态可由 reviewer 升级到 reviewed。

## 7. 下游影响

按 [`01-design-relationships.md`](../01-design-relationships.md) §4.7 与 §4.10 中 G2 的相关边:

```text
A4 → G2
A7 ⇢ G2
G1, G2 → G3
```

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [G3 日志与可观测性设计](G3-logging.md) *(同 Wave 10 sibling)* | → | §4.2 sample time 锁定(G3 logsout 信号采样时戳应与产出端一致);§4.7 调度图(G3 logsout 周期性采样规则);§4.4.2 步骤 8 manifest 字段写入要求 |
| [G1 MIL 顶层结构设计](G1-mil-toplevel.md) *(同 Wave 10 sibling)* | (sibling)— G2 → G3 经 G1 整合 | §4.2 子系统 sample time 设置;§4.3 Rate Transition 块部署位置;§4.4 颜色检查清单 |
| [E5 Controller 性能预算设计](../E-controller/E5-performance-budget.md) *(同 Wave 10 sibling)* | (sibling)| §4.1 SingleTasking 决策;§4.5.2 MultiTasking 升级前置;§4.7 调度图(Controller 5 ms step 间距 = E5 子预算上界) |

间接影响:

- **I4 codegen 配置脚本**(Wave 12):本文件 §4.1 / §4.6 是 I4 在 ert.tlc 配置中"Solver" 与 "Code Generation → Interface" 项的输入。
- **I5 sim runner 脚本**(Wave 12):本文件 §4.4.2 颜色检查程序可由 I5 在自动化模式下通过 Simulink API 复用(`get_param` 读取 `CompiledSampleTime` 替代人工颜色比对)。
- **H3 回归基线策略**:本文件 §4.7 调度图的 20 ms 主窗口决定 H3 基线 .mat 的最小有效采样区间(任何回归对比窗口至少跨 1 个 LCM 主窗口)。

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-10 | G2 author | 初稿 |

## Self-check

- [x] frontmatter 完整,字段值合法
- [x] 上游文档全部存在且 status ≥ reviewed(架构 v1: 2026-05-05;A4: reviewed 2026-05-05;A7: reviewed 2026-05-05;C4: reviewed 2026-05-09;F2: reviewed 2026-05-08)
- [x] 退出条件逐条复核完成,每条均给出依据(§6 共 7 行,3 条退出条件 + 4 条隐含)
- [x] 引用路径全部可点击访问(架构 v1 / A4 / A7 / A6 / C4 / F2 / E5 / G1 / G3 / 01-design-relationships)
- [x] 不存在 RULES §5 禁则中的内容(无 .slx 截图、无可执行 .m;§4.4.2 颜色检查程序为 `pseudo`;无 firmware 实现重述;无 PR/branch 名;无 bus 字段重复定义;依赖列表完整)
- [x] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记 — 本文件 contract_impact=no,N/A
- [x] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径 — 本文件未镜像 firmware 契约,N/A
- [x] 下游影响已沿关系图识别完毕(§7 含 G3 / G1 / E5 + 间接 I4 / I5 / H3)
- [x] 文档不超出本工作项范围(rate-transition 策略与 latency 留 A4;模块周期数值留架构 v1 / A4;harness 顶层拓扑留 G1;logsout 留 G3;Controller 子预算留 E5;Plant 数值积分留 C4;`ins_stub` 内部留 F1/F2;timestamp / dt 留 A7;codegen 完整配置留 I4)
