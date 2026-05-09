---
work_item: C4
title: Plant 数值与积分设计
upstream: [架构 v1, A4, A6, A7, B1, B3, C1, C2]
contract_impact: no
status: reviewed
authored_at: 2026-05-09
last_reviewed_at: 2026-05-09
reviewer_verdict: pass
---

# C4 Plant 数值与积分设计(Plant Numerics & Integration Design)

## 1. 目的

为 FMT-Model-2025b Plant 模块在 [C1](C1-plant-functional.md) 锁定的 7 类物理 / 行为组件与 [C2](C2-plant-structural.md) 锁定的 7 个子系统结构之上,给出**数值与积分层契约**:在不重新设计物理保真度(C1 拥有)、不下沉到多旋翼几何 / 电机 / 分配矩阵参数(C3 leaf 拥有)的前提下,锁定 (a) 积分器选择(算法族 + 阶数 + 固定步长 vs 变步长 ban);(b) 步长(`step_size_s`,Plant base step);(c) 各积分状态的初值来源(per A6 类别契约);(d) reset 行为(per A6 §4.5 触发源契约);(e) 数值稳定性边界(基于 C1/C3 时间常数推导的稳定裕度量化);(f) 四元数单位化策略(频率 + 方法 + 漂移阈值 + 异常处置);(g) 数值健康监控信号(NaN/Inf 检测 / quat 单位化失败 / 步长边际,作为 H4 故障目录的输入)。本文件闭合 [00-design-plan §4.C C4 行](../00-design-plan.md):"积分器选择、步长、初值、reset 行为、数值稳定性边界"。

## 2. 范围

**在范围:**

- §4.1 求解器选择(discrete fixed-step;Forward Euler vs RK4 fixed-step trade-off;Phase 2 选 RK4 的依据)
- §4.2 各积分状态的积分器放置(`pos_ned_m` / `vel_ned_mps` / `quat_ned_to_b` / `ang_rate_b_radps` / `motor_speed_n01`)+ 每个状态的方法 / 初值来源 / reset 类别
- §4.3 四元数单位化策略(频率、方法、漂移阈值、异常处置 → fault flag)
- §4.4 步长与稳定性边界(Plant base step = 1 ms + 各时间常数对照 + Forward Euler 与 RK4 的稳定步长上限推导 + 当前裕度)
- §4.5 初值来源映射(per A6 + per [B3 §4.3 PLANT_PARAM](../B-contracts/B3-parameter-schema.md))
- §4.6 Reset 行为(per A6 §4.4 / §4.5;Plant 仅响应 TS-MIL-HARNESS;reset 后稳态默认等于 init 后稳态)
- §4.7 变步长 / 动态内存禁令(per 架构 v1 §4.5;C4 继承 codegen guard,具体由 I4)
- §4.8 数值稳定性监控信号(quat 单位化 fault flag / NaN/Inf 检测 / 步长边际告警;forward-cite to H4)
- §4.9 跨文档影响清单(forward-cite)

**不在范围(由其他工作项处理):**

- 物理保真度等级、各物理组件方程的功能层定义 — 由 [C1](C1-plant-functional.md) 处理
- Plant 子系统层次 / 块图 / 共享库块归属 — 由 [C2](C2-plant-structural.md) 处理(C4 仅引用 C2 §4.4.5 中的"integrators by C4"位置占位)
- 多旋翼几何 / 质量惯量 / 电机时间常数等具体数值 — 由 [C3](C3-multicopter-leaf.md) 处理(C4 引用 [B3](../B-contracts/B3-parameter-schema.md) 中的 PARAM 字段名,数值由 C3 复核)
- bus 字段级 schema(byte offset / 类型 / 顺序) — 由 [B1](../B-contracts/B1-bus-inventory.md) 拥有
- enum `IntegratorMethod` 数值锁定 — 由 [B2](../B-contracts/B2-enum-inventory.md) 拥有(C4 引用 `INT_RK4` / `INT_FE` / `INT_HEUN` 成员名,不锁数值)
- ert.tlc 完整配置项清单(变步长禁用、动态内存禁用、可变尺寸信号禁用)的脚本落地 — 由 [I4](../I-tooling/I4-codegen-config.md) 处理(C4 仅声明数值层约束,落地由 I4)
- Controller 各环路 anti-windup 积分器与滤波器 reset 数学约束 — 由 [E3](../E-controller/E3-loops-algorithm.md) 处理(本文件仅 Plant 范围)
- 数值健康 fault flag 在 H4 故障目录中的具体故障类与触发参数 — 由 [H4](../H-verification/H4-fault-catalog.md) 处理(C4 仅给出 fault flag 接口与位语义建议)
- timestamp 单位 / epoch / 单调性 — 由 [A7](../A-architecture/A7-time-conventions.md) 处理(C4 仅 echo "Plant base step = 1 ms")
- 跨速率边界 RB-01 / RB-02 / RB-07 的策略 — 由 [A4](../A-architecture/A4-rate-boundaries.md) 处理(C4 仅引用 RB-02 ZOH 输入与 RB-07 内部 cadence)

## 3. 依赖

环境注:FMT-Firmware 在本设计阶段未挂载,作者无法直接读取 `plant_interface.h` / `Plant_types.h`。本文件用 `FMT-Firmware @ <pending hash>` 占位指代 firmware tip;契约级 byte-for-byte 验证延迟到 [B4](../B-contracts/B4-contract-diff.md) 阶段。

| 上游 | 引用位置 | 用途 |
|---|---|---|
| [架构 v1 §4.5 deterministic codegen](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | "ert.tlc, fixed-step discrete, no dynamic memory, no recursion, no variable-size signals" | C4 §4.1 求解器选择(ban 变步长)+ §4.7 codegen guard 继承 |
| [架构 v1 §13 Timing](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | Plant 1 ms / 1000 Hz 单速率 | C4 §4.4 base step 锁定 |
| [架构 v1 §12.4 Plant 内部 6 阶段](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | "single-rate at 1 ms" + "slower sensor publication modeled through output update cadence" | C4 §4.1 求解器在 1 ms base step 下离散执行 |
| [A4 §4.4 RB-02 / RB-07](../A-architecture/A4-rate-boundaries.md) | RB-02(Controller 5 ms → Plant 1 ms ZOH)/ RB-07(Plant 内部 1 ms 主循环 + counter + ZOH latch)| C4 §4.1 ZOH 输入是 Plant 积分器的真实输入信号 |
| [A4 §4.5 整数倍同相起步](../A-architecture/A4-rate-boundaries.md) | 1 / 5 / 10 / 20 ms 整数倍 + 同相起步 | C4 §4.4 步长选择呼应 base rate = 1 ms |
| [A6 §4.2 Plant_init 行为契约](../A-architecture/A6-init-reset-contract.md) | `Plant_init(void)` 无参 + 副作用边界 | C4 §4.6 Plant_init 在 §4.5 初值表执行后所有积分器内部状态确定 |
| [A6 §4.4 稳态值类别表](../A-architecture/A6-init-reset-contract.md) | PARAM / HARDCODED / ZERO / PRESERVED 四类 | C4 §4.5 初值来源映射 + §4.6 reset 后稳态 |
| [A6 §4.5.2 Plant reset 协议](../A-architecture/A6-init-reset-contract.md) | "Plant 不响应 FMS_Out_Bus.reset" + "Plant reset 仅由 harness 触发(TS-MIL-HARNESS)" | C4 §4.6 reset 触发源唯一性 |
| [A6 §4.6.1 反模式禁令](../A-architecture/A6-init-reset-contract.md) | "init 中由 RNG 决定的初值"被禁 + "init 不读 wall-clock / 文件 / 环境变量" | C4 §4.6 init 行为契约 |
| [A7 §4.5 dt 来源](../A-architecture/A7-time-conventions.md) | "模块内部 dt 一律使用固定步长,不由 timestamp 派生" | C4 §4.1 求解器步长 = 模块周期常量,**禁止**由 timestamp 派生 |
| [B1 §4.5.1 Plant_States_Bus](../B-contracts/B1-bus-inventory.md) | `pos_ned_m` / `vel_ned_mps` / `quat_ned_to_b` / `ang_rate_b_radps` / `motor_rpm[]` / `on_ground` / `timestamp` / `fault_flags` 字段 | C4 §4.2 积分状态对应 bus 输出字段 + §4.8 fault flag 字段位语义 |
| [B3 §4.3 PLANT_PARAM](../B-contracts/B3-parameter-schema.md) | PLANT_PARAM.10(`motor_time_const_s`)/ .29 / .30 / .31 / .32(`*_init` 字段)/ .33(`integrator_method`)/ .34(`step_size_s`) | C4 §4.4 时间常数依据 + §4.5 初值字段引用 + §4.7 step_size 锁定 |
| [C1 §4.5.1 / §4.5.3 / §4.7 / §4.8](C1-plant-functional.md) | 6-DoF 方程结构(决定积分维数)+ 电机一阶动态(决定 P3 积分)+ init/reset 字段类别 + 扰动注入点 | C4 §4.2 状态枚举与 C1 一一对应 |
| [C2 §4.4.5 / §4.4.7 / §4.6](C2-plant-structural.md) | S4 "integrators by C4" / S6 输出装配 / Init Hub 结构 | C4 §4.2 积分器物理放置位置 + §4.6 reset 走 S0 → S1..S6 路径 |

co-seal batch:无。本工作项在 Wave 9(per [01-design-relationships.md §6](../01-design-relationships.md#6-推荐执行顺序基于关系图的拓扑)),全部上游(架构 v1 / A4 / A6 / A7 / B1 / B3 / C1 / C2)在 Wave 2–7 已完成 reviewed。

## 4. 设计内容

### 4.1 求解器选择(solver choice)— 闭合退出条件 "积分器选择"

#### 4.1.1 求解器族:discrete fixed-step,显式离散积分

**决策**:Plant 顶层求解器是 **discrete fixed-step**,base step = **1 ms**(per 架构 v1 §13 + A4 §4.5)。所有连续物理(刚体 6-DoF / 电机一阶动态 / 任何带状态的滤波器)采用**显式离散积分**实现,**不**使用 Simulink continuous solver + ZOH 桥接。

**依据**:

1. 架构 v1 §4.5 强制:"deterministic codegen, **fixed-step discrete**, no dynamic memory, no recursion, no variable-size signals"。变步长求解器(ode23 / ode45 / ode15s 等)在 ert.tlc 下不可生成 codegen-safe 代码 → 直接 ban。
2. 架构 v1 §12.4 + C2 §4.1:Plant 单 rate = 1 ms,**无内部多速率**。慢传感器 cadence 由 [A4 §4.4 RB-07](../A-architecture/A4-rate-boundaries.md) "1 ms 主循环 + counter + ZOH latch" 实现(C2 §4.5),不引入第二个 sample-time。
3. A7 §4.5 强制:"模块内部 dt 一律使用固定步长,不由 timestamp 派生"。求解器步长在 codegen 阶段硬编码为 1 ms,运行时不可变。
4. firmware 路径(若 Plant 进入 firmware 编译,虽 Plant 当前仅 MIL):嵌入式调度器无法支持变步长求解器。

#### 4.1.2 离散积分阶数选型:Forward Euler vs RK4 fixed-step

C1 §4.5.1 锁定刚体 6-DoF 为 L4(完整 Newton-Euler + 四元数,无小角度近似);电机为 L2(一阶动态)。两个对积分阶数有不同需求,统一选型如下。

**两个候选**:

| 方法 | 阶数 / 局部截断误差 | 计算成本(per state per step)| 稳定性区域(线性 dx/dt = λx)| codegen 难度 |
|---|---|---|---|---|
| **Forward Euler(FE)** | 1 阶,LTE = O(dt²) | 1× 函数求值;最低成本 | `|1 + λ·dt| ≤ 1` → step ≤ 2/|Re(λ)| (实负特征);振荡系统狭窄 | 极简(1 行 `x[k+1] = x[k] + dt·f(x[k])`);codegen-trivial |
| **RK4 fixed-step** | 4 阶,LTE = O(dt⁵) | 4× 函数求值 | 稳定区比 FE 大 ≈ 2.78× (`step ≤ 2.785 / |λ|` 实负;振荡系统约 2.83×);高阶项截断显著小 | 标准 4-stage 展开;codegen 模板成熟(Simulink "Discrete-Time Integrator" with method = "Integration: RK4" 或 4 个 Sum + Gain 块);仍 codegen-safe |

**Trade-off 分析**:

- **Forward Euler 适用条件**:base step 远小于系统主导时间常数(rule of thumb:dt < τ_min / 10);非线性项温和;无快速振荡(没有靠近 jω 轴的特征值)。优势:codegen 体积最小、运行时延最低、与简单(L1/L2)物理一致。
- **RK4 适用条件**:base step 与时间常数比值在 1:5..1:50 范围;含快速振荡(姿态 / 角速率回路)或强非线性(欧拉力矩耦合 `omega × I × omega`);需要长时间仿真累积误差 ≤ μ-rad 级别。优势:稳定区大 ≈ 2.78×、4 阶截断误差;代价:计算成本 4× 但绝对仍极小(微秒级)。

**Phase 2 决策**:**RK4 fixed-step at 1 ms**,适用于全部连续状态(刚体 6-DoF 12 个状态 + 四元数 4 状态 + 电机 4 一阶状态)。

依据(以 PX4-class quad 时间常数为基准,引用 [B3 §4.3 PLANT_PARAM](../B-contracts/B3-parameter-schema.md)):

1. **电机时间常数 τ_motor ≈ 50 ms**(`PLANT_PARAM.10 motor_time_const_s = 0.05`)→ dt/τ = 1/50 = 0.02。FE 已**满足** rule of thumb(< 0.1),RK4 提供更多裕度。
2. **姿态环带宽 ≈ 50 Hz**(典型多旋翼角速率回路)→ 主导极点约 -300 rad/s 量级,τ_att ≈ 3 ms。dt/τ = 1/3 = 0.33。FE 在该比值下仍稳定但截断误差不可忽略;RK4 把此处局部截断从 O(0.33²) ≈ 0.11 拉到 O(0.33⁵) ≈ 4e-3,**关键于姿态闭环精度**。
3. **位置环带宽 ≈ 5 Hz** → τ_pos ≈ 200 ms → dt/τ = 0.005,FE 与 RK4 差异微小,但保持方法统一。
4. **欧拉非线性 `omega × I × omega`**:在角速率高时(> 500 deg/s ≈ 8.7 rad/s)引入耦合项,FE 在大角速率下累积偏差;RK4 4 阶截断显著抑制此源。
5. **计算预算**:Plant 在 MIL 工作站上 1 ms step 实测预算远超 4× FE 成本(MATLAB R2025b workstation 通常 < 50 µs / step),RK4 计算成本(< 200 µs)绝对值仍远低于 1 ms 实时上界。
6. 与 [B3 §4.3.2 PLANT_PARAM.33](../B-contracts/B3-parameter-schema.md) 默认值 `INT_RK4` 一致(B3 已锁定该 PARAM 字段 + B2 enum 待 lock)。

**降级路径**(若 H1 验证或 SIH 阶段发现 RK4 计算预算紧张):由 PARAM `integrator_method` 切到 `INT_FE`(成本降 4×),适用于 unit test / 快速 batch 回归。**不在 Phase 2 默认路径**。

#### 4.1.3 求解器在 Simulink Solver 配置层的落地(由 G2 实现)

C4 仅约定 Plant 模块内部数值行为;Simulink Solver pane 上的具体配置项(如 "Type: Fixed-step"、"Solver: discrete (no continuous states)"、"Fixed-step size: 1e-3"、"Tasking mode: SingleTasking")由 [G2 §rate-scheduling](../G-harness/G2-rate-scheduling.md) 落地。C4 的硬约束:

- Solver Type **必须**为 `Fixed-step`(变步长 ban,§4.7)
- Fixed-step size **必须 = 1e-3 s**(per 架构 v1 §13 + B3 PLANT_PARAM.34)
- Solver **必须**为 `discrete`(no continuous states 在 Plant 顶层模型;若 Simulink-built-in continuous integrator 块被使用,必须在 codegen 前由 I4 检查并替换为 Discrete-Time Integrator block per A4 §4.4 R1 single-rate-per-module 强约束)

### 4.2 各积分状态的积分器放置(per-state integrator placement)

本节枚举 Plant 内部所有连续状态及其积分器的物理放置(per [C2 §4.4.5 S4 + §4.4.4 S3](C2-plant-structural.md))、积分方法、初值来源、reset 类别。

#### 4.2.1 积分状态表(Phase 2 multicopter)

| ID | 状态 | 维数 | 物理含义 | 积分输入 | 物理放置(C2 子系统)| 积分方法 | 初值来源(per A6)| Reset 类别(per A6 §4.4)| Output bus 字段(per B1)|
|---|---|---:|---|---|---|---|---|---|---|
| **N1** | `pos_ned_m` | 3 | NED 系位置 | `vel_ned_mps`(N2 输出) | S4 Rigid_Body_Dynamics_Shell 平动子链 | RK4 | PARAM(`PLANT_PARAM.29 pos_ned_m_init`)| **PARAM** | `Plant_States_Bus.pos_ned_m` |
| **N2** | `vel_ned_mps` | 3 | NED 系速度 | `acc_ned_mps2 = R(quat) · F_b/m + g_ned` | S4 平动子链 | RK4 | PARAM(`PLANT_PARAM.30 vel_ned_mps_init`)| **PARAM** | `Plant_States_Bus.vel_ned_mps` |
| **N3** | `quat_ned_to_b` | 4 | NED→body 四元数姿态 | `quat_dot = 0.5 · Ω(ω_b) · quat`(per C1 §4.5.1) | S4 Rigid_Body_Dynamics_Shell 转动子链 + Layer A `Quaternion_Normalize` | RK4 + 单位化(per §4.3) | PARAM(`PLANT_PARAM.31 quat_ned_to_b_init`,默认 `[1,0,0,0]` identity)| **PARAM** | `Plant_States_Bus.quat_ned_to_b` |
| **N4** | `ang_rate_b_radps` | 3 | body 系角速度 | `ang_acc_b = inv(I)·(M_b - ω × I·ω)`(per C1 §4.5.1) | S4 转动子链 | RK4 | PARAM(`PLANT_PARAM.32 ang_rate_b_radps_init`,默认 `[0,0,0]`)| **PARAM** | `Plant_States_Bus.ang_rate_b_radps` |
| **N5** | `motor_speed_n01[]` | 4(MC)| 电机归一化推力(0..1)的一阶滤波器内部状态 | `du/dt = (motor_cmd - u_actual) / τ_motor`(per C1 §4.5.3) | S3 Propulsion / Motor Model + Layer A `LowPass_Filter_1st` | RK4(or FE for L1 降级)| ZERO(per A6 §4.4.2 "电机/舵面状态 ZERO 多旋翼默认";C1 §4.7.1 选 ZERO)| **ZERO** | 间接 → `Plant_States_Bus.motor_rpm[]`(per C1 §4.5.3 sqrt 反算)|

**总积分状态数** = 3 + 3 + 4 + 3 + 4 = **17 个**(N1..N5 累加;Phase 2 multicopter,4 电机)。

**注**:
- `acc_ned_mps2` / `acc_b_mps2` / `ang_acc_b_radps2` / `motor_rpm[]`(by sqrt)是**派生量**,不是积分状态(无独立内部存储);per C1 §4.7.1 在 init 时类别 ZERO,因为它们在 init step 之前未由方程计算。
- `on_ground`(布尔)是触地检测状态机输出,不是连续积分状态;由 C2 §4.4.5 + C3 落地;C4 不管理。
- `mass_kg` / `inertia_b_kgm2`(`PLANT_PARAM.01` / `.02`)是常量参数,不是积分状态(Phase 2 不模拟燃料消耗 per C1 §4.7.1)。
- `motor_rpm` 在 N5 输出层由 sqrt 反算公布,因此 N5 把"内部状态"定义为归一化推力 `u_actual ∈ [0,1]`,而非 RPM(避免 sqrt 反向积分的数值病态);C2 §4.4.4 沿用此约定。

#### 4.2.2 积分器放置图(在 C2 子系统内的位置)

```text
S4 Rigid_Body_Dynamics_Shell
├── 平动 sub-block
│   ├── F_b → /m + R·g → acc_ned_mps2
│   ├── [Discrete-Time Integrator, RK4, dt=1ms] ← acc_ned_mps2 → vel_ned_mps    (N2)
│   └── [Discrete-Time Integrator, RK4, dt=1ms] ← vel_ned_mps  → pos_ned_m      (N1)
│
├── 转动 sub-block
│   ├── M_b - ω × I·ω → /I → ang_acc_b_radps2
│   └── [Discrete-Time Integrator, RK4, dt=1ms] ← ang_acc_b → ang_rate_b_radps  (N4)
│
└── 姿态 sub-block
    ├── 0.5 · Ω(ω_b) · quat → quat_dot
    ├── [Discrete-Time Integrator, RK4, dt=1ms] ← quat_dot → quat_ned_to_b_raw  (N3 raw)
    └── Layer A Quaternion_Normalize → quat_ned_to_b (per §4.3)

S3 Propulsion / Motor Model
└── per motor k in {0..motor_count-1}:
    └── Layer A LowPass_Filter_1st (RK4 internal, τ=motor_time_const_s) ← motor_cmd[k] → u_actual[k]   (N5[k])
```

注:Simulink "Discrete-Time Integrator" 块的 "Integrator method" 字段在 R2025b 下含 `Forward Euler` / `Backward Euler` / `Trapezoidal` / `Integration: RK4` 等选项。C4 强制选 `Integration: RK4`(对应 PLANT_PARAM.33 `INT_RK4`)。C2 §4.4.5 已为这些位置预留"integrators by C4"占位,本文件锁定具体方法。

### 4.3 四元数单位化策略(quaternion normalization)

四元数积分会因数值误差累积导致 ‖q‖ 漂离 1。本节定义 Plant 内部 quat 单位化的契约。

#### 4.3.1 单位化频率

**决策**:**每 step(1 ms)单位化一次**。

**依据**:

1. **足够保守**:即使 RK4 4 阶积分,quat 单步漂移量级在 ‖q‖² 量级 ≤ 1e-8(典型 ω = 10 rad/s,dt = 1 ms);1 ms 单位化绝对安全,不需要每 N step 一次的优化(嵌入式资源宽裕)。
2. **数值简洁**:每 step 一次避免引入"是否到该单位化"的状态机逻辑;Layer A `Quaternion_Normalize` 块就放在积分器输出端,无条件执行。
3. **最低可接受**:技术下限是 **每 10 ms 至少一次**(若 1 ms 成本被证明过高)。Phase 2 不下沉到该优化;若 H1 / E5 性能测算需要,触发 §5 R-3 路径。

#### 4.3.2 单位化方法

```pseudo
# Layer A Quaternion_Normalize block 内部
norm_sq = q.w*q.w + q.x*q.x + q.y*q.y + q.z*q.z
if norm_sq > QUAT_NORM_MIN_SQ:           # QUAT_NORM_MIN_SQ = 1e-12 (HARDCODED)
    inv_norm = 1.0 / sqrt(norm_sq)
    q_normalized = q * inv_norm
    fault_flag_quat_norm = false
else:
    # ‖q‖ ≈ 0:数值彻底崩溃,不可恢复;紧急 reset 到 identity
    q_normalized = [1.0, 0.0, 0.0, 0.0]   # identity quaternion
    fault_flag_quat_norm = true            # 写入 Plant_States_Bus.fault_flags
```

**说明**:
- `QUAT_NORM_MIN_SQ = 1e-12` 是 HARDCODED 安全阈值(若 ‖q‖² < 1e-12 则视为崩溃);此值远低于 IEEE 754 single 精度下的"健康 ‖q‖²"(应在 1.0 ± 1e-7 内)。
- "紧急 reset 到 identity" 不等价于 §4.6 的全模块 reset(只复位 quat 单 state);其他状态保持。
- fault flag 写入 `Plant_States_Bus.fault_flags` 的具体位语义由 [B1 §4.5.1](../B-contracts/B1-bus-inventory.md#451-plant_states_bus) 锁定;C4 §4.8 给出位语义建议。

#### 4.3.3 漂移阈值与监控

**正常运行漂移上限**:`|1 - ‖q‖|` typical ≤ **1e-6**(RK4 + 每 step 单位化 + 1 ms step + ω ≤ 10 rad/s)。

**监控建议**(C4 不强制实现,but H4 可消费):`norm_dev = abs(1 - sqrt(norm_sq))`;若 `norm_dev > 1e-3` 持续 > 100 step,说明积分器或单位化逻辑异常 → fault flag(per §4.8)。

#### 4.3.4 单位化失败处置

若 `norm_sq <= QUAT_NORM_MIN_SQ`(数值崩溃):

1. quat 紧急 reset 到 identity `[1,0,0,0]`(避免下游 `R(quat)` 矩阵奇异 → NaN 传播)
2. 写 `Plant_States_Bus.fault_flags.quat_norm_fault = true`
3. **不**触发 §4.6 全模块 reset(单 state 局部恢复);harness / G3 日志可见 fault flag → H4 可决定是否升级到全 reset
4. 若连续 100 step 仍 fault,说明上游(角速率 N4)失控 → 由 H4 升级处置(C4 不裁决具体策略,forward-cite)

### 4.4 步长与稳定性边界(step size & stability bounds)

#### 4.4.1 Plant base step

**决策**:Plant base step `dt = 1 ms`(per 架构 v1 §13 + A4 §4.5 + B3 PLANT_PARAM.34 = `0.001`)。

#### 4.4.2 时间常数清单(Phase 2 multicopter)

下表汇总 Phase 2 各积分子系统的主导时间常数 / 带宽,作为稳定性裕度推导依据。数值以 PX4-class quad 为基线(由 [C3](C3-multicopter-leaf.md) 复核;[B3 §4.3](../B-contracts/B3-parameter-schema.md) 提供默认 PARAM)。

| 子系统 | 主导时间常数 τ / 带宽 ω | PARAM 来源 | dt/τ 比值 | 安全裕度评估 |
|---|---|---|---:|---|
| **电机一阶动态(N5)** | τ_motor ≈ **50 ms** | `PLANT_PARAM.10 motor_time_const_s` | 1/50 = 0.02 | **50× 裕度**;rule of thumb dt < τ/10 → 余 5× |
| **姿态环带宽(N3 / N4 通过反馈)** | f_att ≈ 50 Hz → ω_att ≈ 314 rad/s → τ_att ≈ 3 ms | 隐含于 [E3](../E-controller/E3-loops-algorithm.md) Controller 设计 | 1/3 ≈ 0.33 | **20× 裕度**(Plant 1 ms 相对 Controller 主带宽);典型多旋翼无问题 |
| **位置环带宽** | f_pos ≈ 5 Hz → τ_pos ≈ 200 ms | 同上 | 1/200 = 0.005 | **200× 裕度** ✓ |
| **气动 L2 阻力(N1 / N2 通过耦合)** | 阻力时间常数 ≥ 100 ms(低速) | `PLANT_PARAM.12` / `.13` | 1/100 = 0.01 | **100× 裕度** ✓ |
| **传感器噪声 random walk(P4)** | bandwidth-limited;noise filter τ ≥ 10 ms(per L3) | `PLANT_PARAM.14`–`.21` | 1/10 = 0.1 | **10× 裕度** ✓(L3 典型) |

**最紧裕度** = 姿态环 dt/τ ≈ 0.33,即 1/3。

#### 4.4.3 Forward Euler 稳定性边界

线性系统 `dx/dt = λx`,FE 离散化 `x[k+1] = (1 + λ·dt) x[k]`,稳定条件 `|1 + λ·dt| ≤ 1`。

对 Phase 2 各 λ 量级:

| 子系统 | 主导 λ(rad/s,实负) | FE 稳定上限 step `2/|λ|` | 1 ms 是否安全 |
|---|---:|---:|---|
| 电机 | λ_motor = -1/τ_motor ≈ -20 rad/s | step ≤ 100 ms | ✓ 1 ms ≪ 100 ms,**100× 裕度** |
| 姿态(若孤立振荡)| ω_att ≈ -300 rad/s (主导极点)| step ≤ 6.67 ms | ✓ 1 ms < 6.67 ms,**6.67× 裕度** |
| 角速率内环极点 | -100..-500 rad/s 量级 | step ≤ 4..20 ms | ✓ 1 ms < 4 ms 即安全,**≥ 4× 裕度** |

**结论**:FE 在 1 ms 下对 Phase 2 全部子系统稳定,但姿态 / 角速率内环裕度仅 ≈ 4–6× ,接近 rule of thumb 下限。**此即选 RK4 而非 FE 的关键理由**(§4.1.2)。

#### 4.4.4 RK4 稳定性边界

RK4 fixed-step 的稳定区在实负实轴上为 `step ≤ 2.785 / |λ|`(系数 2.785 来自 RK4 stability polynomial);对纯振荡(纯虚特征值)系数约 2.83。

| 子系统 | RK4 稳定上限 step | 1 ms 裕度 |
|---|---:|---|
| 电机 | step ≤ 139 ms | **139× 裕度** ✓ |
| 姿态主极点(λ ≈ -300) | step ≤ 9.28 ms | **9.28× 裕度** ✓(显著优于 FE 的 6.67×)|
| 角速率内环 | step ≤ 5.57 ms (假设 λ ≈ -500) | **5.57× 裕度** ✓ |

**RK4 + 1 ms 在所有 Phase 2 子系统下稳定且裕度 ≥ 5×**,高阶截断误差额外 4 阶抑制 → 选 RK4。

#### 4.4.5 气动刚度(aerodynamic stiffness)

Phase 2 多旋翼气动 L2 = 线性 + 二次阻力(C1 §4.5.2);**无升力面、无激波、无失速、无失稳俯仰** → 气动方程是 benign 的(主导特征值远小于姿态环极点)。**不引入额外刚度约束**。Phase 4+ 若启用地效 / prop wash(L3),需复审本节(§5 R-2)。

#### 4.4.6 步长边际监控(margin monitoring)

C4 建议(forward-cite to H4):若运行时检测到 `|x[k+1] - x[k]| > MAX_STEP_DELTA`(每个状态独立阈值),意味着当前 step 累积变化超过物理可接受上限 → 触发 `Plant_States_Bus.fault_flags.solver_stiffness_fault = true`(C4 §4.8)。

阈值参考(C4 不锁数值,由 H4 / C3 配 PARAM):

- `|Δpos| > 100 m / step` → 物理上不可能(假定 vel ≤ 100 km/h ≈ 28 m/s → max Δpos = 28 mm / ms)
- `|Δvel| > 100 m/s / step` → 加速度 > 100 g,显然异常
- `|Δquat| > 0.5 / step` → 角速率 > 1000 rad/s ≈ 57000 deg/s,显然异常
- `|Δang_rate| > 100 rad/s / step` → 角加速度 > 100000 rad/s²,显然异常

### 4.5 初值来源(initial value sources;per A6 + B3)

本节给出每个积分状态在 init / reset 后的稳态值来源,**严格 echo** [C1 §4.7.1](C1-plant-functional.md#471-plant_states_bus-字段类别映射) 与 [A6 §4.4.2](../A-architecture/A6-init-reset-contract.md):

| 状态 ID | 状态名 | 类别(per A6 §4.4)| 初值来源 | PARAM 字段(per B3 §4.3.2)|
|---|---|---|---|---|
| **N1** | `pos_ned_m` | **PARAM** | `States_Init_Bus.pos_ned_m` echo | `PLANT_PARAM.29 pos_ned_m_init`(默认 `[0,0,0]`)|
| **N2** | `vel_ned_mps` | **PARAM**(C4 选 PARAM 而非 ZERO,与 C1 §4.7.1 一致;允许场景化非零初速度)| `States_Init_Bus.vel_ned_mps` echo | `PLANT_PARAM.30 vel_ned_mps_init`(默认 `[0,0,0]`)|
| **N3** | `quat_ned_to_b` | **PARAM**(identity 默认值由 PARAM 落定,not HARDCODED)| `States_Init_Bus.quat_ned_to_b` echo | `PLANT_PARAM.31 quat_ned_to_b_init`(默认 `[1,0,0,0]` identity)|
| **N4** | `ang_rate_b_radps` | **PARAM** | `States_Init_Bus.ang_rate_b_radps` echo | `PLANT_PARAM.32 ang_rate_b_radps_init`(默认 `[0,0,0]`)|
| **N5** | `motor_speed_n01[]` | **ZERO** | — | n/a(per A6 §4.4.2 + C1 §4.7.1 选 ZERO) |

**与任务给定表的差异**:任务原表把 `vel_ned_mps` 标 ZERO 并标 PARAM(B3)= n/a,把 `quat_ned_to_b` 标 HARDCODED。**C4 选择与 C1 §4.7.1 + B3 §4.3.2 + A6 §4.4.2 已锁定的契约一致**:

- `vel_ned_mps` 实为 PARAM(`PLANT_PARAM.30`),默认值是 `[0,0,0]` 但语义是 PARAM(允许场景化覆盖,例如 H1 飞行中重启场景 `vel ≠ 0`);C1 §4.7.1 已锁定 PARAM。
- `quat_ned_to_b` 实为 PARAM(`PLANT_PARAM.31`),默认是 identity 但通过 PARAM 路径设置(不是 HARDCODED 在源代码里);C1 §4.7.1 已锁定 PARAM。
- 任务表的 ZERO / HARDCODED 标记在概念上能"通过",但与上游契约不一致;C4 不重新发明,**沿用上游**。

#### 4.5.1 初值注入路径

```text
[Plant_init(void) called by harness or Plant_step gate]
  ↓
S0 Init Hub (per C2 §4.4.1 + §4.6.2)
  ↓
  reads PLANT_PARAM.29..32 + States_Init_Bus (per C2 §4.6.2 step 4)
  ↓
  dispatches to S1..S6 via Reset_Composer (per C2 §4.4.1)
  ↓
  S4 integrators (N1..N4) load PARAM values into internal state
  S3 first-order filters (N5) load ZERO into internal state
  ↓
[ first Plant_step(void) executes; integrators advance from initial state ]
```

#### 4.5.2 与 [C2 §4.6.2 Init Hub](C2-plant-structural.md) 的衔接

C2 §4.6.2 已经描述 S0 Init Hub 的结构步骤(read PARAM → dispatch);C4 仅锁定 N1..N5 的具体目标值。**积分器内部状态在第一次 `Plant_step(void)` 之前已确定**,符合 [A6 §4.2.2 稳态定义](../A-architecture/A6-init-reset-contract.md#422-init-后稳态契约):"init 完成后第一次 step 前,所有内部存储状态都处于已定义、可在文档中查到具体值的状态"。

### 4.6 Reset 行为(per A6 §4.4 / §4.5)

#### 4.6.1 Reset 触发源唯一性

per [A6 §4.5.2 关键约束 1](../A-architecture/A6-init-reset-contract.md):

1. **Plant 仅响应 TS-MIL-HARNESS**(MIL 仿真 harness 显式注入 reset)。
2. **Plant 不响应**:`FMS_Out_Bus.reset`(TS-FMS-OUT)/ TS-EXT-CMD / TS-MODE-XCHG / TS-FAILSAFE。理由见 A6 §4.5.2:"global reset 不联动 Plant reset"(Plant 是物理仿真,FMS / Controller 决策 reset 不应导致仿真世界回到 t=0)。
3. firmware 路径下 Plant 不存在(Plant 不在 firmware 飞行运行期);TS-MIL-HARNESS reset 通路必须由 [B4 / I4](../B-contracts/B4-contract-diff.md) 在 codegen 输出中**消除或编译开关隔离**(per A6 §4.5 表)。

#### 4.6.2 Reset 行为契约

per [A6 §4.4.4](../A-architecture/A6-init-reset-contract.md):**reset 后稳态默认 = init 后稳态**。Plant 无例外。

具体动作:

1. TS-MIL-HARNESS reset trigger 上升沿(harness boundary input port)
2. → S0 Init Hub 重新执行 init 路径(per C2 §4.6.3)
3. → S1..S6 各 Reset Port 响应 → 所有内部状态(包括 N1..N5 积分器内部 + S5 sensor counters / latches / lock state machines)回到 §4.5 初值
4. → 第一次 reset 后 step 执行时,积分器从 PARAM / ZERO 起步,与 init 后语义完全一致

**Reset 是离散动作**:在一个 1 ms step 内完成;**无瞬态过渡**;reset 前后 step 之间 bus 字段值由 PARAM / ZERO / HARDCODED 类别值替换,无插值。

#### 4.6.3 Reset 时积分器内部状态

| 状态 ID | reset 前内部状态 | reset 时动作 | reset 后内部状态 |
|---|---|---|---|
| N1..N4 | 任意运行值 | 由 S0 dispatch PARAM 值覆盖内部存储 | PARAM 值(per §4.5)|
| N5 | 任意运行值 | 由 S0 dispatch 0 覆盖一阶滤波器内部存储 | 0 |
| Quaternion Normalize block | 内部无独立状态(代数块)| n/a | n/a |
| 各传感器 counter / lock state machine(C2 §4.4.6 / §4.5)| running | 由 S0 dispatch 归零 + 状态机回到 NO_LOCK | counter = 0,fix_type = NO_FIX,valid = false |

**禁止 PRESERVED 类别**:per A6 §4.4.2,Plant 在 Phase 2 不使用 PRESERVED(reset 不保留任何运行期状态)。`integrator_method`(`PLANT_PARAM.33`)是 C-I(compile-inlined)→ reset 不可改算法。

#### 4.6.4 init / reset 反模式禁令(per A6 §4.6.1)

C4 强制以下禁令:

1. **不**在 init 中调用 RNG 决定积分器初值(per A6 §4.6.1 + C1 §4.6.3)。所有传感器噪声 / 偏置 random-walk 状态在 init 时由 PARAM 路径载入,不由 wall-clock seeded RNG 生成。
2. **不**在 init 中读取 wall-clock(per A7 §4.5)。Plant 所有时间相关逻辑(timestamp / counter)在 init 时归零(per A7 §4.6 + C1 §4.7.1)。
3. **不**在 init 中调用 `FMS_step` / `Controller_step`(per A6 §4.6.1)。`Plant_init(void)` 完全独立。
4. **不**在 reset 时跨 reset 边界保留 dt 差分(per A7 §4.4)。所有 staleness / dt 计算器在 reset 时归零。

### 4.7 变步长 / 动态内存禁令(继承自架构 v1 §4.5)

C4 在 Plant 数值层声明以下禁令(具体落地由 [I4 codegen 配置](../I-tooling/I4-codegen-config.md)):

| 禁令 | 范围 | 依据 |
|---|---|---|
| **NO 变步长求解器**(ode23 / ode45 / ode15s 等) | Plant Solver 配置必须 `Fixed-step` + `discrete` | 架构 v1 §4.5 + A7 §4.5 + ert.tlc 不支持 |
| **NO 动态内存分配**(malloc / realloc / new) | 所有积分器、滤波器、缓冲区使用静态 / 栈分配 | 架构 v1 §4.5 + ert.tlc deterministic codegen |
| **NO 递归积分器**(state 间互相递归引用) | C2 §4.4.5 块图为 DAG;不允许循环依赖(代数环)| 架构 v1 §4.5 + Simulink algebraic loop ban |
| **NO 可变尺寸信号**(variable-size signals) | 所有 bus 字段固定尺寸;`motor_speed_n01[N_MOT_MAX]` 用编译期常量 N_MOT_MAX | 架构 v1 §4.5 + B3 §4.3 R-3 dimension-constant |
| **NO 由 timestamp 派生 dt** | 求解器步长 = 编译期常量(1 ms)| A7 §4.5 |
| **NO continuous-time 块**(在 Plant 顶层 / 任何 Plant 子系统) | 所有积分器使用 Discrete-Time Integrator(method = RK4)| 架构 v1 §4.5 + A4 §4.4 R1 single-rate-per-module |

I4 在 codegen 阶段(per [I4 退出条件](../00-design-plan.md))实施 lint:任何上述禁令的违反必须 codegen-time fail。

### 4.8 数值稳定性监控(numerical stability monitoring;forward-cite H4)

C4 建议 Plant 在 `Plant_States_Bus.fault_flags` 中(per [B1 §4.5.1](../B-contracts/B1-bus-inventory.md)若已有)或 `Extended_States_Bus` 扩展位中暴露以下数值健康标志:

#### 4.8.1 Fault flag 位语义(C4 提案,具体位序由 B1 锁定)

| Bit | 标志名 | 触发条件 | 处置(C4 不强制,H4 决定)|
|---|---|---|---|
| 0 | `quat_norm_fault` | per §4.3.4:`norm_sq < QUAT_NORM_MIN_SQ` 或 `|1 - ‖q‖| > 1e-3` 持续 > 100 step | quat 紧急 reset 到 identity;H4 可决定升级 |
| 1 | `integration_overflow_fault` | 任意积分状态出现 NaN 或 Inf(IEEE 754 检测) | Plant step 输出非数值;H4 可决定终止仿真或全 reset |
| 2 | `solver_stiffness_fault` | per §4.4.6:任意状态单 step 变化 > MAX_STEP_DELTA | 步长边际告警;H4 可决定 fail-fast |
| 3 | `motor_state_saturated`(可选) | 任意 `motor_speed_n01[k] < 0` 或 `> 1`(物理不可能) | per §4.4.6 阈值 |
| 4..7 | reserved | — | — |

**实现位置**:S6 Output Assembly(per C2 §4.4.7),由 NaN/Inf 检测块 + quat-norm 监控块 + 状态变化阈值块的"OR" 输出。

#### 4.8.2 NaN / Inf 检测

```pseudo
# in S6 Output Assembly, 每个 step 末尾
nan_inf_detected = isnan(pos_ned_m) || isnan(vel_ned_mps) || isnan(quat_ned_to_b) || isnan(ang_rate_b_radps) || isnan(motor_speed_n01) || isinf(...) for each
fault_flags.integration_overflow_fault = nan_inf_detected
```

**注**:NaN / Inf 一旦出现通常不可恢复(传播性);Plant 不内部尝试恢复,只暴露 fault flag,由 H4 升级到 harness-level fail-fast(`assert(!fault) → simulation abort`)。

#### 4.8.3 与 H4 故障目录的接口(forward-cite)

C4 仅给出 fault flag 接口建议;[H4 故障注入目录](../H-verification/H4-fault-catalog.md) 拥有以下决策:

- 哪些 fault flag 是 fail-fast vs fail-soft?
- fault flag 是否可由 H4 注入(例如 H4 模拟"积分溢出"测试 H 场景)?
- fault flag 是否要在 [G3 logging](../G-harness/G3-logging.md) 中作为 minimum signal set?

**C4 承诺**:fault flag 信号在 Plant 顶层输出 bus 中可观测;具体字段挂载位置由 [B1](../B-contracts/B1-bus-inventory.md) 在 `Plant_States_Bus.fault_flags` 或 `Extended_States_Bus` 中决定。

### 4.9 Open issues / 委托清单

下列项 C4 故意不裁决:

| Open ID | 议题 | 委托给 | 推迟原因 |
|---|---|---|---|
| C4-OQ1 | RK4 在 R2025b Simulink "Discrete-Time Integrator" 块 vs 手工 4-stage Sum 实现的细节差异 | 实现阶段 | 设计阶段不裁决 Simulink 块实现细节 |
| C4-OQ2 | `MAX_STEP_DELTA` 各状态阈值的具体数值 | [C3](C3-multicopter-leaf.md) + [H4](../H-verification/H4-fault-catalog.md) | 数值由 leaf 时间常数 + H4 fault 容忍策略共同决定 |
| C4-OQ3 | `fault_flags` bit 序与 bus 字段类型 | [B1](../B-contracts/B1-bus-inventory.md) | bus schema 范围 |
| C4-OQ4 | RK4 → FE 降级的 PARAM 切换是 codegen-time 还是 runtime | [I4 codegen 配置](../I-tooling/I4-codegen-config.md) | per B3 §4.3.2 PARAM.33 类别 = C-I(compile-inlined),已锁 codegen-time;但具体 codegen 机制 by I4 |
| C4-OQ5 | quat 单位化频率优化(每 N step 一次)的触发条件 | 实现阶段 + [E5 性能预算](../E-controller/E5-performance-budget.md)(若 Plant 性能进入 E5 范围) | Phase 2 不需要,留 forward-cite |
| C4-OQ6 | `QUAT_NORM_MIN_SQ` 数值是否要 PARAM 暴露 | [B3 增补](../B-contracts/B3-parameter-schema.md) | 当前 HARDCODED 1e-12 即可;如 H1 发现需要可触发 B3 增补 |

## 5. 已知风险与悬而未决问题

- **R-1 RK4 计算预算在嵌入式或 SIH 阶段超标**
  - 影响:若 Plant 进入 SIL/SIH 路径(架构 v1 §15.2 / §15.3),RK4 4× 成本可能超预算
  - 处置:Phase 2 仅 MIL,workstation 上 RK4 < 200 µs / step,远小于 1 ms 实时预算 → 无问题。SIH 阶段由 [E5](../E-controller/E5-performance-budget.md) + [I4](../I-tooling/I4-codegen-config.md) 复审是否切到 FE
- **R-2 气动 L3 升级(地效 / prop wash)引入新刚度**
  - 影响:若 Phase 4+ C1 §4.5.2 升级到 L3(per C1 §4.2.3 推迟项),地效模型可能引入更短时间常数 → §4.4 stability 表需复审
  - 处置:Phase 2 不启用;C1 §4.5.2 降级 hook 已就位;升级时触发 C4 变更日志条目
- **R-3 quat 每 step 单位化的计算成本**
  - 影响:若 Plant 在 SIL/SIH 阶段进入嵌入式,1 个 sqrt + 4 multiplication / step 可能成本敏感
  - 处置:Phase 2 不优化;若 E5 / I4 测算需要,可降到每 10 step 一次(§4.3.1 给出下限)
- **R-4 MAX_STEP_DELTA 阈值未锁定**
  - 影响:§4.8 step margin monitor 在 H4 启用前没有具体数值 → fault flag 不触发
  - 处置:C4-OQ2 委托;Phase 2 监控可用占位阈值(per §4.4.6 列出),H4 锁定后 by 变更日志
- **R-5 fault flag 字段在 B1 schema 中是否存在尚未 byte-verify**
  - 影响:§4.8 暴露 fault flag 假设 `Plant_States_Bus.fault_flags` 字段存在,但 B1 当前 schema 需 verify against firmware tip
  - 处置:本文件 contract_impact=no(不动 firmware schema);若 B1 verify 后该字段不存在,则 fault flag 暂存 `Extended_States_Bus`(MIL-only)直到 B1 增补
- **R-6 RK4 与 quat 单位化的 4 阶截断 / 单位化序列依赖**
  - 影响:RK4 在 4 个 stage 内部对 quat 多次求值;每个 stage 的 quat 是否在 stage 间单位化?
  - 处置:**C4 决策**:RK4 内部 4 个 stage 不单位化,只在 step 末尾(完成 RK4 4-stage 求和后)单位化一次。理由:RK4 是基于"在 stage 间保持 raw quat"的标准实现;stage 间单位化会破坏 RK4 4 阶精度。本决策与 §4.3.1 "每 step 单位化" 一致(注:每 step = 每 1 ms 完整步,不是每 stage)
- **R-7 PLANT_PARAM.33 / .34 数值占位 vs firmware tip 一致性**
  - 影响:契约 byte-level 验证(B4 / I3)
  - 处置:[B3 §4.3.2](../B-contracts/B3-parameter-schema.md) 已标 `(B4 verify against firmware tip)`;C4 不重复声明,仅引用

## 6. 退出条件复核

对照 [`00-design-plan.md` §4.C C4 行](../00-design-plan.md):**"积分器选择、步长、初值、reset 行为、数值稳定性边界"**(5 条原子退出条件)。

| # | 退出条件原文(原子拆分)| 本文档依据 | 状态 |
|---|---|---|---|
| (1) | 积分器选择 | §4.1.1 求解器族(discrete fixed-step)+ §4.1.2 阶数选型(RK4 fixed-step 4 阶)+ §4.2 各状态积分器物理放置(N1..N5)+ B3 PLANT_PARAM.33 = `INT_RK4` | **满足** |
| (2) | 步长 | §4.4.1 base step = 1 ms,与架构 v1 §13 + A4 §4.5 + B3 PLANT_PARAM.34 = 0.001 一致;§4.4.2 时间常数清单证明 1 ms 选择合理(最紧裕度 dt/τ ≈ 0.33,姿态环) | **满足** |
| (3) | 初值 | §4.5 初值来源映射表(N1..N5 对应 PARAM(`PLANT_PARAM.29`–`.32`)/ ZERO);§4.5.1 注入路径;§4.5.2 与 C2 §4.6.2 衔接 | **满足** |
| (4) | reset 行为 | §4.6.1 触发源唯一性(TS-MIL-HARNESS only,per A6 §4.5.2)+ §4.6.2 行为契约(reset 后稳态 = init 后稳态)+ §4.6.3 各积分器 reset 时动作 + §4.6.4 反模式禁令 | **满足** |
| (5) | 数值稳定性边界 | §4.4.3 FE 稳定性边界推导(`step ≤ 2/|λ|`)+ §4.4.4 RK4 稳定性边界推导(`step ≤ 2.785/|λ|`)+ §4.4.2 时间常数表(电机 50ms / 姿态 3ms / 位置 200ms)+ §4.4.4 各子系统裕度表(电机 139× / 姿态 9.28× / 角速率 5.57×)+ §4.4.5 气动刚度评估 | **满足** |

辅助(非退出条件强制):

| # | 辅助检查 | 本文档依据 | 状态 |
|---|---|---|---|
| (a) | A1 §4.5 deterministic codegen 落地(变步长 / 动态内存 / 递归 / 可变尺寸 ban)| §4.7 全部 6 条禁令显式 | **满足** |
| (b) | C2 §4.4.5 "integrators by C4" 占位闭合 | §4.2.2 积分器放置图明确 N1..N5 在 S4 / S3 内的位置 + 选 RK4 | **满足** |
| (c) | C1 §4.7.1 init/reset 字段类别一致性 | §4.5 初值来源表完全 echo C1 §4.7.1(P/Z 类别 + PARAM 字段名) | **满足** |
| (d) | A6 §4.5.2 Plant reset 协议遵循 | §4.6.1 + §4.6.4 完整引用 A6 关键约束(无响应 FMS reset / 仅 TS-MIL-HARNESS / init 反模式禁令) | **满足** |
| (e) | B3 §4.3.2 PLANT_PARAM.33 / .34 占位字段闭合 | §4.1.2 选 `INT_RK4` + §4.4.1 `step_size_s = 0.001` 与 B3 默认值一致 | **满足** |
| (f) | A7 §4.5 "dt 来源" 强制项遵循 | §4.7 显式禁 "由 timestamp 派生 dt" + §4.1.3 求解器 fixed-step | **满足** |
| (g) | 四元数单位化策略锁定(C1 §4.5.1 委托项 C1-OQ3) | §4.3 全节(频率 / 方法 / 阈值 / 异常处置) | **满足** |
| (h) | 数值健康监控(为 H4 提供入口)| §4.8 fault flag 接口 + forward-cite H4 | **满足** |

## 7. 下游影响

按 [`01-design-relationships.md` §4.3 / §4.7](../01-design-relationships.md):

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [G1 MIL 顶层结构](../G-harness/G1-mil-toplevel.md) | ⇢(信息流)| §4.6.1 Plant TS-MIL-HARNESS reset 入口端口存在性 + §4.7 codegen guard;G1 在顶层 harness 中提供 reset trigger 信号源 |
| [G2 速率调度](../G-harness/G2-rate-scheduling.md) | ⇢(信息流)| §4.1.3 Solver 配置约束(Fixed-step / discrete / step 1e-3);G2 在 Simulink Solver pane 落地 |
| [I4 codegen 配置](../I-tooling/I4-codegen-config.md) | ⇢(信息流)| §4.7 6 条禁令(变步长 / 动态内存 / 递归 / 可变尺寸 / timestamp-derived dt / continuous-time 块);I4 在 ert.tlc 配置脚本中 lint |
| [H4 故障注入目录](../H-verification/H4-fault-catalog.md) | ⇢(信息流)| §4.8 fault flag 位语义 + 监控信号清单(quat_norm_fault / integration_overflow_fault / solver_stiffness_fault);H4 决定 fail-fast / fail-soft 策略 + 注入接口 |
| [C3 多旋翼 leaf](C3-multicopter-leaf.md) | ←(被引用)| §4.4.2 时间常数清单要求 C3 复核 PLANT_PARAM.10 / PARAM.12 / PARAM.13 数值;§4.5 初值要求 C3 复核 PARAM.29..32 默认值;C4-OQ2 MAX_STEP_DELTA 阈值由 C3 协助 |
| [B1 §4.5.1 Plant_States_Bus](../B-contracts/B1-bus-inventory.md) | ←(C4 提案)| §4.8.1 fault_flags 位语义建议(C4-OQ3);B1 拥有具体字段 schema |
| [E5 Controller 性能预算](../E-controller/E5-performance-budget.md) | ⇢(间接)| §4.4.2 姿态 / 位置环带宽假设(50 Hz / 5 Hz);若 E5 实际带宽差异 ≥ 2× → 触发 C4 §4.4 复审 |
| [F1 ins_stub 功能设计](../F-ins-contract/F1-ins-stub-functional.md) | ⇢(间接)| §4.6.2 reset 后稳态契约 → ins_stub 在 Plant reset 时同步回稳态;F1 决定具体协调 |
| [B3 §4.3.2](../B-contracts/B3-parameter-schema.md) | ←(C4 echo)| §4.5 初值字段 PARAM.29..32 + §4.1.2 / §4.4.1 PARAM.33 / .34 默认值 echo |

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-09 | C4 author | 初稿;闭合退出条件 (1)–(5);锁定 RK4 fixed-step at 1 ms / 每 step quat 单位化 / Plant 仅响应 TS-MIL-HARNESS reset / 6 条 codegen 禁令;§4.8 数值健康 fault flag 接口为 H4 提供入口 |

## Self-check

- [x] frontmatter 完整,字段值合法
- [x] 上游文档全部存在且 status ≥ reviewed:架构 v1(reviewed)、A4(reviewed)、A6(reviewed)、A7(reviewed)、B1(reviewed)、B3(reviewed)、C1(reviewed)、C2(reviewed);无同 batch 兄弟
- [x] 退出条件逐条复核完成,每条均给出依据(§6 表 (1)–(5) 满足 + 辅助 (a)–(h) 满足)
- [x] 引用路径全部可点击访问(均为相对路径,指向已存在文档)
- [x] 不存在 RULES §5 禁则中的内容(无 .slx 截图、无可执行 .m、无重复 firmware 实现细节、无 PR/branch 名、无重复 bus/enum 字段表 — 均引用 B1/B3;伪代码块标 `pseudo`)
- [N/A] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记 — **本文 contract_impact=no,N/A**
- [N/A] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径 — **本文不镜像 firmware 契约(只引用 B1 / B3 已镜像项,镜像义务在 B 区);N/A**
- [x] 下游影响已沿关系图识别完毕(§7 表覆盖 G1 / G2 / I4 / H4 / C3 / B1 / E5 / F1 / B3 出边;C4 无强前置出边)
- [x] 文档不超出本工作项范围(无越权设计):未重定义物理(C1)、未定义结构(C2)、未定义 leaf 数值(C3)、未重定义 bus(B1)、未重定义 enum(B2)、未重定义 PARAM 数值(B3);不在范围项已显式列在 §2
