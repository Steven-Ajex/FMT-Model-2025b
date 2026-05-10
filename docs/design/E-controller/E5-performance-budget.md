---
work_item: E5
title: Controller 性能预算设计
upstream: [架构v1, A4, A7, B3, E1, E2, E3, E4]
contract_impact: no
status: draft
authored_at: 2026-05-10
last_reviewed_at:
reviewer_verdict: none
---

# E5 Controller 性能预算设计

## 1. 目的

为 Controller 在 5 ms 周期内的执行行为给出**可预测、可验证的预算**:对位置 / 速度 / 姿态 / 角速度 / mixer 五大环路 + 总线装配 + reset/init 逻辑分配 µs 级时间预算,显式列出**热点路径上禁用**的 Simulink 构造(MATLAB Function、可变尺寸信号、动态内存等),给出查找表(LUT)使用规则与 Phase 2 浮点 / Phase 5+ 定点策略框架,并界定 profiling 与跨 Wave 协同(G2 SingleTasking / H1 timing realism)的接入面。本文件**不**重定义任一环路的算法(由 E3 拥有)、级联结构(由 E2 拥有)、多旋翼 leaf 的 mixer 矩阵(由 E4 拥有)或 cmd_mask 裁剪规则(由 E1 拥有);仅在以上设计已锁定的前提下,**对实现阶段的执行时间与构造选用做规则约束**。

## 2. 范围

**在范围:**
- Controller 5 ms / 200 Hz 周期内 §4.1 的 per-loop µs 级时间预算分配(Phase 2 多旋翼 baseline)
- 热点路径(L-01..L-04 + Mixer)上禁用的 Simulink / Stateflow 构造清单与允许替代
- LUT(查找表)使用规则:维度上限、网格大小预算、插值方式、storage class、Phase 2 候选清单
- Phase 2 浮点策略声明(`single` 全程)+ Phase 5+ 定点策略占位框架
- E3 滤波器 / anti-windup 已锁定的算法对预算的贡献核算
- Profiling / verification 接入点(MIL profiler / codegen `cycle_counter` / H1 timing-realism 场景)
- 1300 µs reserve 的去向规则(变体度量 / 未来 Phase 3+ feature / G2 MultiTasking 迁移)

**不在范围(由其他工作项处理):**
- 各环路的控制律 / 前馈 / anti-windup / 滤波器**算法**定义 — 由 [E3 各环算法设计](E3-loops-algorithm.md) 处理
- 级联拓扑 / shared-vs-leaf / 共享库块认领 — 由 [E2 Controller 结构设计](E2-controller-structural.md) 处理
- 多旋翼 mixer 矩阵 / 电机推力曲线 / leaf 增益结构 — 由 [E4 Controller 多旋翼 leaf 设计](E4-multicopter-leaf.md) 处理
- cmd_mask 裁剪规则 — 由 [E1 Controller 功能设计](E1-controller-functional.md) 处理
- Solver 配置 / Sample Time 颜色检查 / 多速率调度 — 由 G2 多速率调度设计处理(Wave 10 sibling)
- MIL 顶层 .slx 拓扑 — 由 G1 MIL 顶层结构设计处理(Wave 10 sibling)
- ert.tlc 配置项实现 / lint 规则细节 — 由 [I4 Codegen 配置脚本设计](../I-tooling/I4-codegen-config.md)(待启动)处理
- 验证场景具体 datapack / 阈值 — 由 [H1 验证场景目录](../H-verification/H1-scenario-catalog.md)(待启动)处理

## 3. 依赖

| 上游 | 引用位置 | 用途 |
|---|---|---|
| 架构 v1 §4.5 | [架构 v1](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | "deterministic codegen: ert.tlc, fixed-step discrete, no dynamic memory, no recursion, no variable-size signals" — 热点禁用项的根源条款 |
| 架构 v1 §13 | [架构 v1](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | Controller 5 ms / 200 Hz, internally single-rate;§13.3 "Controller execution budget is most critical; architecture should minimize bus slicing and heavy MATLAB Function use in that path" — 总预算与 hot-path 约束的来源 |
| 架构 v1 §17 risk 5 | [架构 v1](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | "Timing realism risk: MIL may look stable while scheduler/topic/timestamp behavior in firmware or SIH reveals coupling bugs" — verification 接入策略的根源 |
| A4 跨速率边界 | [A4 跨速率边界设计](../A-architecture/A4-rate-boundaries.md) | Controller 端 5 ms 单速率;Plant 1ms / FMS 20ms / INS 10ms 在 Controller 边界处由 ZOH/latch 处理,Controller 内部不再细分速率 |
| A7 时间约定 §4.5 / §4.8 | [A7 时间与时间戳约定](../A-architecture/A7-time-conventions.md) | Controller dt = 固定步长 5 ms,**非** `timestamp` 派生;timestamp 仅用于 staleness/log/profiling — 决定 dt 不进入算法热路径 |
| B3 §4.5 CONTROL_PARAM | [B3 Parameter schema](../B-contracts/B3-parameter-schema.md) | CONTROL_PARAM.29 `rate_d_lpf_cutoff_hz`、.36 `thrust_factor`、.40-.44 `K_aw_*`(2026-05-09 amendment)的 R-T 分类与默认值;C-I vs R-T 决定 codegen storage class,从而影响 LUT inlining |
| E1 §4 cmd_mask 裁剪规则 | [E1 Controller 功能设计](E1-controller-functional.md) | 哪些环在哪些 mode 下被裁剪 → 决定 worst-case 路径的环路启用集合;E5 取 worst-case = 全部 5 环全开 |
| E2 §4.1 顶层拓扑 | [E2 Controller 结构设计](E2-controller-structural.md) | 7 子系统(Setpoint Decode / Position / Velocity / Attitude / Rate / Mixer / Output Assembler)→ 预算行项目对齐 |
| E3 §4.1..§4.8 算法定义 | [E3 各环算法设计](E3-loops-algorithm.md) | per-loop 算法步骤数(乘 / 加 / 滤波器更新)→ 估计每环 µs 上界;§4.6 back-calculation anti-windup 与 §4.7 D-LPF 的额外计算成本 |
| E4 §4.2 4×4 mixer 逆 | [E4 Controller 多旋翼 leaf 设计](E4-multicopter-leaf.md) | quad-X mixer = 4×4 矩阵乘 → Mixer 行预算 |

## 4. 设计内容

### 4.1 5 ms 周期预算分配(Phase 2 多旋翼 baseline)

**总预算** = Controller period = 5 ms = **5000 µs**(per 架构 v1 §13、A4、E2 §4.1)。

每环 / 每行项目的预算如下表。预算单位 µs;百分比相对于 5000 µs 总额。

| # | 行项目 | 预算 (µs) | 占比 | per-cycle 关键算子 | 算法依据 |
|---|---|---:|---:|---|---|
| 1 | L-01 Position loop | 200 | 4.0% | P-only;3-axis;轻量 | E3 §4.2 |
| 2 | L-02 Velocity loop | 500 | 10.0% | PI + 重力前馈 + back-calc + 测量 LPF;3-axis | E3 §4.3 / §4.6.1 / §4.7.1 |
| 3 | L-03 Attitude loop | 600 | 12.0% | quaternion error + P;quat 数学较重(quat conjugate / multiply / slerp-free 路径) | E3 §4.4 |
| 4 | L-04 Angular rate loop | 1500 | 30.0% | PID + D-LPF(强制) + back-calc;3-axis hot path,Controller 最贵的环 | E3 §4.5 / §4.6.2 / §4.7.2 / §4.7.3 |
| 5 | Mixer / Allocator | 200 | 4.0% | 4×4 线性分配矩阵 + thrust_factor 标量补偿 | E4 §4.2 |
| 6 | Setpoint Decode + Bus assembly + I/O | 500 | 10.0% | cmd_mask 解码 / Bus Creator / Outport;两侧外圈 | E1 / E2 §4.1.1 + §4.1.7 |
| 7 | Reset / init 逻辑 | 200 | 4.0% | per-step init checks(integrator 清零判定 / quat identity 初始化判定 / D-LPF 状态判定) | A6 + E3 §4.9 |
| 8 | Reserve / margin | 1300 | 26.0% | 测量方差吸收 + 未来 feature 余量 | 见 §4.9 |
| | **Total** | **5000** | **100%** | | |

**释义**:
- 1300 µs reserve(26%)显著高于"安全余量"经验下限(典型 RTOS hot-loop 推荐 ≥ 20%),目的是吸收 (a) MIL→codegen→host SIL→target HIL 间不可避免的执行时间方差;(b) Phase 3+ feature 增长(轨迹跟踪 FF / 气动补偿);(c) 未来 G2 由 SingleTasking 迁移至 MultiTasking 时的上下文切换开销。
- L-04 占 30% 是有意识的:它是 200 Hz 全速运行的最内环、PID + D-LPF 同时作用、且每周期都不被 cmd_mask 裁剪(per E1 典型 always-on 假设)。
- L-03 quaternion 数学(误差四元数 + axis-angle 提取 + body-frame 投影)单 cycle 估计 ≈ 30 浮点乘 + 20 浮点加 + 1 平方根,在 Cortex-M4F @ 168 MHz 单精度 FPU 下 ≈ 100 cycle ≈ 0.6 µs;预算 600 µs 留出 1000× 头量,足以容纳任何 reasonable 实现而不被工具链优化噪声击穿。

**worst-case 取数原则**:本表按"全部 5 环都开"取 worst case(即 stabilize/posctl/full-cascade 的 superset);裁剪后(例如 throttle passthrough mode 仅 mixer + assembler 跑)实际执行时间会显著低于 5000 µs,reserve 自然变大。

**何时重测预算**:
1. 本表数值在以下任一变更时必须重新测量并更新:
   - Phase 3+ 新增 feature 引入新行项目(如轨迹 FF / 气动补偿)
   - G2 由 SingleTasking 切换至 MultiTasking
   - Controller 周期由 5 ms 调整(此为契约级变更,需走 INDEX 决策日志)
   - target MCU 架构变更(如由 Cortex-M4F 迁移至 Cortex-M7 / Cortex-A 系)

### 4.2 热点路径禁用项

热点路径定义 = L-01 / L-02 / L-03 / L-04 / Mixer 五个 §4.1 行项目;Setpoint Decode / Bus Assembly / Reset 不属于热点路径但仍须遵守第 1-5 项(避免不确定性扩散)。

| # | 禁用构造 | 根源 | 理由 | 替代做法 |
|---|---|---|---|---|
| 1 | MATLAB Function block(在热点路径) | 架构 v1 §13.3 | MATLAB Function 在 ert.tlc 下虽生成 C,但其内部 MATLAB 语义会引入难以静态界定的循环 / 动态分支 / 隐式内存占用;且 lint 工具难以从生成 C 反推原始约束。架构 v1 §13.3 显式建议 minimize | (a) **Embedded MATLAB Function**(Stateflow Embedded MATLAB)+ `coder.unroll` 显式展开定长循环;(b) Simulink 原生块(Sum / Product / Gain / Lookup1D / Lookup2D / Saturation / DiscreteFilter)组合;(c) 若需复用,封装为 Library Subsystem |
| 2 | 任意 unbounded loop | 架构 v1 §4.5 "no recursion / no variable-size" 衍生 | 不定长循环 → 不可静态界定 worst-case 执行时间 | For-Each Subsystem with **fixed bounded N**(per-axis 迭代 N=3;per-motor 迭代 N=4);或 `coder.unroll` 显式展开 |
| 3 | 递归(任何形式) | 架构 v1 §4.5 | 函数调用栈深度不定 → MCU 栈耗尽风险 | 等价的迭代结构,N 静态可界 |
| 4 | 可变尺寸信号 | 架构 v1 §4.5 | 触发 codegen 动态内存或附加 size 字段,违反 hot-path 静态性 | 固定 N 维向量(Vec3 / Quat / Vec4 motor);需要"长度可变"语义时,改用 fixed 上限 N + valid-mask |
| 5 | 动态内存分配(`malloc` / `realloc` / `free`) | 架构 v1 §4.5 | RTOS 实时性破坏;碎片化风险 | 静态分配 + Simulink Bus + Outport |
| 6 | String / cell / char 数组(在热点路径) | 架构 v1 §4.5 衍生 | 不定长 + 比较代价不可界 | Enum(per B2);整数 ID;固定长度 char[N] 仅用于日志/状态行 |
| 7 | Stateflow 中的 MATLAB action(超过定长展开) | 架构 v1 §13.3 衍生 | 等价于隐藏的 MATLAB Function | 改用 Simulink 块或 Stateflow Graphical Function |
| 8 | `tic` / `toc` / `clock` / 任何获取 wall-clock 的调用(in algorithm) | A7 §4.5 / §4.8 | dt 固定步长来源,timestamp 不进入算法 | 算法层假设 dt = sample_time(常量 5 ms);profiling 采集走外置 cycle counter,不进算法 |
| 9 | 浮点比较 `==` / `!=`(在条件分支) | numerics 安全 | 精度抖动会改变路径 → 测量噪声 | 用 `>= eps` 或 `<= eps` 容忍带 |
| 10 | 大尺寸 lookup(超 §4.3 上限)| §4.3 | cache miss 风险 | 拆分为多个小 LUT;或改用解析公式 |

**lint 强制点**:本表 1-6 + 8 项作为 [I4 Codegen 配置脚本](../I-tooling/I4-codegen-config.md) 的 lint 规则输入(forward-cite Wave 12);I4 启动时 E5 §4.2 的列表作为锁定输入。

### 4.3 LUT(查找表)策略

#### 4.3.1 何时使用 LUT

仅当满足以下**全部**条件,才允许在热点路径上使用 LUT:

1. 函数解析形式过重(单 cycle 估算 > 20 µs,即与本环预算量级相当)且
2. 函数定义域可静态界定且
3. 在该域内函数光滑(C¹ 连续),线性插值的最大误差可接受(典型 < 1%)且
4. 网格大小不超过 §4.3.2 上限。

不满足上述条件的,优先用解析公式 + Simulink 原生块。

#### 4.3.2 LUT 网格 / 维度 / 插值规则

| 维度 | 上限 | 备注 |
|---|---|---|
| 1D LUT 单轴元素数 | ≤ 64 | 64 × 4 byte single = 256 byte,L1 cache 友好 |
| 2D LUT 每轴元素数 | ≤ 32 | 32×32 × 4 byte = 4 KB |
| 3D LUT | **禁用** | 需要 3D 时改为公式或两个 2D 级联 |
| 插值方法 | linear(双线性 for 2D) | 不允许 cubic / spline(单 cycle 不可静态界定) |
| 边界处理 | clip(saturate at endpoints) | 不允许 extrapolation |
| storage class | const compile-inlined(per [B3 §4.5 R-2 / R-4](../B-contracts/B3-parameter-schema.md)) | 由 codegen 内联为 ROM 常量;支持 MCU flash 直接 fetch |

#### 4.3.3 Phase 2 LUT 候选(占位 — 实现阶段决定是否真正落表)

| 候选 | 形式 | 大小 | 理由 | 当前判断 |
|---|---|---|---|---|
| `quat_from_thrust_vector_yaw(thrust_dir_norm, yaw_setpoint)` | 2D LUT | 32×32 = 1024 元素 | E3 §4.8.4 thrust-vector cascade 中,从 (thrust_dir, yaw) 推导 attitude reference 的解析路径含 atan2 / cross / quat-from-axis-angle,> 30 浮点运算 | **暂不用 LUT**(解析路径在 600 µs L-03 预算内可承受);留作 Phase 3+ 优化窗 |
| 电机推力曲线非线性补偿 | 1D LUT | 32 元素 | E4 motor 推力非线性 | **暂不用 LUT**(per [B3 CONTROL_PARAM.36 `thrust_factor`](../B-contracts/B3-parameter-schema.md),Phase 2 baseline 用单参数线性近似;Phase 3+ 升级为 LUT 时本节激活) |

Phase 2 baseline = **零 LUT 在热点路径**(全解析 + 原生块)。本节先行声明上限规则,Phase 3+ 上 LUT 时直接套用。

### 4.4 浮点 / 定点策略

#### 4.4.1 Phase 2 声明:全 `single`(float32)

Phase 2 multicopter MIL 路径 **全部热点路径信号 = `single` (float32)**。理由:

1. 与 [B3 §4.5 CONTROL_PARAM](../B-contracts/B3-parameter-schema.md) 全部增益 / 限幅 / cutoff 字段类型一致(`single`,4 byte)→ 无类型转换开销
2. 与 [B1 Bus](../B-contracts/B1-bus-inventory.md) Vec3 / Quat / Euler 子 schema 类型一致
3. Cortex-M4F 单精度 FPU 单 cycle 完成 mul/add → MCU FLOPS 充裕

**FLOPS 估算**:5 个 hot-path 环,每周期合计粗估 ~30 floating-ops × 200 Hz = 6 kFLOPs/s;再乘以 3-axis 展开 ≈ 18 kFLOPs/s。Cortex-M4F @ 168 MHz 名义 FPU 峰值 168 MFLOPs/s → 占用 < 0.02%(此估算仅含算术运算,memory 访问与控制流不计入,但仍说明 FPU 端预算极松)。

#### 4.4.2 Phase 5+ 定点策略框架(占位)

若 / 当 firmware target 切换至无 FPU 的 Cortex-M0+ / DSP / 自定义芯片时,本节升级为定点设计。占位规则:

| 信号类 | 备选格式 | 理由 |
|---|---|---|
| 角度 / 角速度 | Q15 saturating(`fixdt(1, 16, 12)`) | 范围 ±8 rad / ±8 rad/s 充裕,12 bit 小数位精度 ~ 0.00024 rad |
| 位置 / 速度 | Q31 saturating(`fixdt(1, 32, 16)`) | 32-bit 字长保证大 range(地面坐标 km 量级) |
| PID 增益 | Q15(`fixdt(1, 16, 14)`) | 增益 < 4 |
| 中间累加器 | 64-bit accumulator,饱和后再截 | 防中间溢出 |
| LUT 输出 | 与下游一致 | 避免类型转换 |

Phase 2 不实现以上;仅声明"如何切"的方向,以备 Phase 5+ 启动时不重新设计。

### 4.5 Profiling 与验证接入

per 架构 v1 §17 risk 5 timing realism + RULES §6 可验证性:

#### 4.5.1 三阶段 profiling

| 阶段 | 工具 | 测量对象 | 通过判据 |
|---|---|---|---|
| MIL(模型层) | Simulink Profiler | 每子系统 wall time / cycle count | 每行项目 ≤ §4.1 预算 |
| Codegen 后 host SIL | 生成 `.c` 中插入 `cycle_counter` 宏(由 [I4 Codegen 配置](../I-tooling/I4-codegen-config.md) 添加 hooks) | 每子系统 host CPU cycles | 与 MIL 测量一致(相对偏差 < 30%) |
| Pre-merge gate | H1 timing-realism 场景 | end-to-end Controller_step 实测 max | ≤ 5000 µs;reserve 利用率 ≤ 100% |

#### 4.5.2 H1 timing-realism 场景(forward-cite)

Controller 性能预算的最终验证场景由 [H1 验证场景目录](../H-verification/H1-scenario-catalog.md)(待启动)拥有;E5 在此声明对 H1 的输入要求:

- 至少一个场景跑全 5 环开启路径(stabilize → posctl → loiter)的 ≥ 60 s 长度仿真;
- profiling 关闭对照组(不开 `cycle_counter`)+ profiling 开启对比组,验证 profiling 自身开销 ≤ 5%;
- max-cycle-time + 99-th percentile cycle time 双指标判据:
  - max ≤ 5000 µs(absolute deadline)
  - 99-th ≤ 4000 µs(20% margin 仍在 reserve 内)

H1 启动时本 §4.5.2 作为 `H1 ⇢ E5` 的输入条款。

### 4.6 G2 SingleTasking 协同(Wave 10 sibling)

[G2 多速率调度设计](../G-harness/G2-rate-scheduling.md)(Wave 10 sibling,co-seal-eligible)拥有 Solver 配置与 SingleTasking / MultiTasking 选型。E5 在此声明本预算所基于的调度假设:

1. **SingleTasking 模式**:Controller 全部 5 ms 子系统在同一 task 上下文按 §4.1 顺序串行执行。本预算表的累加是直接的 wall-time 和。
2. **task overrun 处置**:若实际执行时间 > 5000 µs,Simulink Profiler 报 warning;codegen 后由 RTOS 层捕获;判为 H1 timing-realism 场景失败。
3. **MultiTasking 推迟**:Phase 2 不切 MultiTasking;若 Phase 5+ 切,则 L-04 hot path 可能拆到更高优先级 task,届时 §4.1 预算重写 + reserve 重新分配。
4. **任务边界处约定**(per A4 §4):FMS → Controller(20 ms → 5 ms)用 ZOH;Plant → Controller via INS_Out_Bus(10 ms → 5 ms)用 ZOH;Controller → Plant(5 ms → 1 ms)用 ZOH。这些 rate transition 块的执行不计入 §4.1 而由 Bus Assembly + I/O 行(500 µs)吸收(rate transition 在 codegen 中通常 < 1 µs / 通道)。

G2 评审时 E5 §4.6 作为 `E5 ⇢ G2` 的输入(Wave 10 内同 batch 信息流,无强前置)。

### 4.7 滤波器 / D-LPF 对预算的贡献

per [E3 §4.7.3](E3-loops-algorithm.md)(L-04 D 项 LPF,**强制**)+ [B3 CONTROL_PARAM.29 `rate_d_lpf_cutoff_hz`](../B-contracts/B3-parameter-schema.md)(默认 30 Hz,Phase 2 三轴共用):

- 单极 Tustin 离散低通 = 3 浮点乘 + 2 浮点加 + 1 状态保存 → Cortex-M4F 单 cycle ≈ 4 cycles ≈ 0.024 µs / 通道
- 三轴共用 cutoff 但状态独立 → 3 × 0.024 = 0.072 µs / 周期
- cutoff 在 [30, 50] Hz 范围内调节不改变运算量(只改系数)→ 调参不影响预算
- E5 结论:D-LPF 在 L-04 1500 µs 预算内占 < 0.01%,可忽略;cutoff 范围调整无预算压力。

群延迟影响(per E3 §4.7.3):cutoff 30 Hz @ 200 Hz 采样 → 群延迟 ~5 ms,与 L-04 周期同量级。这是控制律设计层面的 trade-off,不是 E5 预算层面的关注点。

### 4.8 Anti-windup 计算成本核算

per [E3 §4.6 back-calculation](E3-loops-algorithm.md) + [B3 CONTROL_PARAM.40-.44 `K_aw_*`](../B-contracts/B3-parameter-schema.md)(2026-05-09 sealed amendment):

- back-calc 单轴单环 = 1 浮点乘 + 1 浮点减 + 1 加到积分器 = 3 ops / cycle
- L-02 velocity loop:3 axes(xy 共用 K_aw_vel_xy + z 用 K_aw_vel_z)= 3 × 3 = 9 ops
- L-04 rate loop:3 axes(roll/pitch/yaw 各用 K_aw_rate_*)= 3 × 3 = 9 ops
- 合计 18 ops / 周期 ≈ 0.1 µs / 周期(Cortex-M4F)
- 在 L-02 500 µs 与 L-04 1500 µs 预算内合计占 < 0.01%

E3 §4.6 的双层保护(back-calc 主路径 + `*_int_lim_*` hard-clamp)中,clamp 部分仅一个 saturate 块,< 1 cycle / axis,合计 < 0.1 µs。

E5 结论:anti-windup 不构成预算压力;实现时不需特殊优化。

### 4.9 Reserve / margin(1300 µs)预设去向

26% reserve 不是无主预算,事先按以下优先级分配:

| 优先级 | 用途 | 预设额度 (µs) | 触发条件 |
|---|---|---:|---|
| P-1 | B4 contract diff & MIL→codegen 测量方差 | 200 | 始终保留(已被实际占用,不可重新分配) |
| P-2 | Phase 3+ 轨迹跟踪 yaw-to-thrust-vector FF | 200 | Phase 3 启动 |
| P-3 | Phase 3+ 气动 / 风扰补偿(若 Plant 升 L3 fidelity) | 300 | C 系列升 fidelity |
| P-4 | G2 由 SingleTasking 切 MultiTasking 上下文切换开销 | 100 | Phase 5+ MultiTasking 启用 |
| P-5 | 通用 slack(未规划用途) | 500 | reserve buffer |
| | **合计** | **1300** | |

reserve **不允许**用于:
- 算法层临时优化欠缺(应该改算法,不蚕食 reserve)
- 未经 H1 验证的 feature 堆叠
- profiling overhead(profiling 必须额外 budget)

reserve 重新分配 = E5 文档变更 = 触发 [01-design-relationships.md §7](../01-design-relationships.md) 变更回溯规则,沿 `E5 ⇢ G2` / `E5 ⇢ H1` 出边通知下游。

### 4.10 与下游 / 同 Wave 的交叉引用

- [E1 Controller 功能设计](E1-controller-functional.md)(强前置 ⇢):cmd_mask 裁剪规则决定 worst-case 环路启用集合;E5 取"全部 5 环全开"作为 worst case(superset of E1 任一 mode 的实际启用)
- [E2 Controller 结构设计](E2-controller-structural.md):§4.1.1..§4.1.7 七子系统对齐 §4.1 预算行项目
- [E3 各环算法设计](E3-loops-algorithm.md)(强前置 →):算法粒度决定 §4.1 每行的 µs 估计;§4.7 滤波器 / §4.6 anti-windup 计算成本由 §4.7-§4.8 核算
- [E4 Controller 多旋翼 leaf 设计](E4-multicopter-leaf.md):4×4 mixer 矩阵 = §4.1 行 5 的算子来源
- [A4 跨速率边界](../A-architecture/A4-rate-boundaries.md):rate transition(20→5 / 10→5 / 5→1)开销由 Bus Assembly + I/O 行吸收
- [A7 时间约定 §4.5 / §4.8](../A-architecture/A7-time-conventions.md):dt = 固定 5 ms,timestamp 不进算法
- [B3 Parameter schema](../B-contracts/B3-parameter-schema.md):CONTROL_PARAM.29 D-LPF cutoff;.36 thrust_factor;.40-.44 K_aw_*
- [G1 MIL 顶层](../G-harness/G1-mil-toplevel.md)(Wave 10 sibling):承载 Controller 在 MIL 顶层的接入;E5 §4.5 profiler 在此运行
- [G2 多速率调度](../G-harness/G2-rate-scheduling.md)(Wave 10 sibling):Solver / SingleTasking 选型与本预算的累加假设一致
- [I4 Codegen 配置](../I-tooling/I4-codegen-config.md)(forward-cite,Wave 12):§4.2 禁用项作为 lint 规则输入;§4.3 LUT storage class 作为 codegen 配置输入
- [H1 验证场景目录](../H-verification/H1-scenario-catalog.md)(forward-cite,Wave 11):timing-realism 场景的 max-cycle-time + 99-th percentile 双指标判据

## 5. 已知风险与悬而未决问题

- **R-1 MIL→codegen→host SIL→target HIL 测量方差未实测**
  - 影响:§4.1 表中 µs 数为基于 Cortex-M4F @ 168 MHz 单精度 FPU 的设计估计,未经实际 HIL 验证。Phase 2 MIL 阶段最多能验证 §4.5.1 三阶段中的前两阶段;target HIL 验证要等 SIH/HIL 设计阶段(架构 v1 Phase 3+)。
  - 处置:reserve 预先吸收(P-1 200 µs);H1 timing-realism 场景在 MIL 阶段先暴露大方差,target HIL 启动时再补一轮预算修订(变更回溯沿 E5 ⇢ G2 / H1 出边)。

- **R-2 §4.1 µs 估计未做 worst-case 静态分析**
  - 影响:本表是基于"reasonable 实现 + 100× 头量"的设计预算,不是 WCET(Worst-Case Execution Time)分析。极端测量噪声 / cache miss / branch misprediction 可能超 §4.1 单行预算,但仍在 5000 µs 总额内(因 reserve 26%)。
  - 处置:认可这是 design-time 估算;实际收敛由 §4.5 的三阶段 profiling 闭环提供。WCET 静态分析工具(如 aiT)Phase 2 不引入(成本与 Phase 2 目标不匹配)。

- **R-3 LUT 候选清单(§4.3.3)Phase 2 全部 deferred**
  - 影响:E5 给出 LUT 上限规则但 Phase 2 不实际使用,规则的实操检验留到 Phase 3+。
  - 处置:open;Phase 3+ 轨迹 FF 启动时 §4.3.3 候选 1 自动激活;§4.3.2 上限作为 lint 规则即刻生效。

- **R-4 定点策略(§4.4.2)未细化**
  - 影响:Phase 5+ 切定点时本节需要展开为完整设计;现仅占位框架。
  - 处置:open;Phase 5+ target MCU 选定后启动定点设计(可能新增 E6 工作项,或扩展 E5 §4.4.2)。

- **R-5 G2 / H1 同 Wave 内信息流(`E5 ⇢ G2` / `E5 ⇢ H1`)目前是 forward-cite**
  - 影响:本文件的 §4.5.2 / §4.6 引用 H1 / G2 的内容是设计意图占位,H1/G2 启动时可能修订其内部约定,届时 E5 需回溯。
  - 处置:Wave 10/11 启动时 `E5 ⇢ G2` 与 `E5 ⇢ H1` 走同 batch / 邻近 batch 信息共享;允许同 batch 兄弟互相引用(per RULES §6)。

## 6. 退出条件复核

对照 [`00-design-plan.md`](../00-design-plan.md) §4.E 中本工作项 E5 的退出条件:

> 退出条件原文:**"5 ms 周期内各环执行预算、热点禁用项(MATLAB Function 等)、查找表与定点策略"**

| # | 退出条件分项 | 本文档依据 | 状态 |
|---|---|---|---|
| 1 | 5 ms 周期内各环执行预算 | §4.1 — 7 行项目 + 1 reserve 共 5000 µs;每行附 µs 数 / 占比 / per-cycle 算子 / 算法依据 + §4.7 滤波器贡献 + §4.8 anti-windup 贡献 + §4.9 reserve 去向 | 满足 |
| 2 | 热点禁用项(MATLAB Function 等)| §4.2 — 10 项禁用清单;每项给出根源(架构 v1 §4.5 / §13.3 / A7)+ 理由 + 替代做法;forward-cite I4 lint 强制 | 满足 |
| 3 | 查找表策略 | §4.3 — 触发条件 / 维度上限 / 网格大小 / 插值方法 / 边界处理 / storage class;§4.3.3 Phase 2 候选清单(均 deferred)| 满足 |
| 4 | 定点策略 | §4.4 — Phase 2 全 `single` 浮点(声明 + FLOPS 估算);§4.4.2 Phase 5+ 定点占位框架(信号类 / 备选格式 / 理由)| 满足 |

## 7. 下游影响

按 [`01-design-relationships.md`](../01-design-relationships.md) 本工作项 E5 的出边:

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| G1 MIL 顶层(Wave 10 sibling) | (sibling 信息流) | Controller 在 MIL 顶层的 wall-time deadline = 5000 µs;profiler 接入需求(§4.5.1 三阶段) |
| G2 多速率调度(Wave 10 sibling) | (sibling 信息流;§4.6) | SingleTasking 假设;rate transition 在 Bus Assembly 行吸收;Phase 5+ MultiTasking 切换的 reserve 余量 P-4(100 µs) |
| H1 验证场景目录(Wave 11) | forward-cite(`E5 ⇢ H1`) | timing-realism 场景需 ≥ 60s 全 5 环开启路径 + max ≤ 5000 µs / 99-th ≤ 4000 µs 双指标判据(§4.5.2) |
| I4 Codegen 配置(Wave 12) | forward-cite | §4.2 禁用项 1-6 + 8 → lint 规则;§4.3 LUT storage class = const compile-inlined → codegen 配置;§4.4.1 hot-path 全 `single` → 数据类型策略 |

**触及 firmware 契约**:`contract_impact: no`。本文件不新增 / 修改任何 bus / enum / parameter / EXPORT 字段;仅约束模型仓侧的实现选择。已用到的 `K_aw_*`(.40-.44)与 `rate_d_lpf_cutoff_hz`(.29)字段均由 [B3 sealed-amendment(2026-05-09)](../B-contracts/B3-parameter-schema.md) 拥有,E5 仅引用,不重定义。

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-10 | author | 初稿 |

## Self-check

- [x] frontmatter 完整,字段值合法
- [x] 上游文档全部存在且 status ≥ reviewed(架构 v1 / A4 / A7 / B3 / E1 / E2 / E3 / E4 均已 reviewed,见 INDEX 已完成清单);G1/G2/H1/I4 是 forward-cite,在 §4.10 / §7 显式标注 Wave 10/11/12,不构成 §3 必备前置
- [x] 退出条件逐条复核完成,每条均给出依据(§6 共 4 行覆盖原文 4 个分项)
- [x] 引用路径全部可点击访问
- [x] 不存在 RULES §5 禁则中的内容(无 .slx 截图、无可执行 .m、无 firmware 实现细节复述、无 PR 名占位、无 bus/enum 重复定义、无凭空依赖、无 firmware 契约镜像 — 仅 forward 引用 B3 既有字段)
- [N/A] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记 — `contract_impact=no`,不适用
- [N/A] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径 — 本文不镜像任何 firmware 契约,不适用
- [x] 下游影响已沿关系图识别完毕(§7 含 4 个下游;§4.10 给出 11 项交叉引用)
- [x] 文档不超出本工作项范围(无越权设计;§2 显式列出 6 项不在范围;算法 / 结构 / leaf / cmd_mask / Solver / 验证场景细节均交由 E3 / E2 / E4 / E1 / G2 / H1 处理)
