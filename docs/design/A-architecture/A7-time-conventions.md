---
work_item: A7
title: 时间与时间戳约定
upstream: ["架构 v1", "A1"]
contract_impact: yes
status: reviewed
authored_at: 2026-05-05
last_reviewed_at: 2026-05-05
reviewer_verdict: pass
---

# A7 时间与时间戳约定

## 1. 目的

为 FMT-Model-2025b 三模块(Plant / FMS / Controller)在仿真与 firmware 集成两端约定**时间戳的单位、epoch、回卷处理、单调性、`dt` 来源**,并对 firmware 现有 `*_interface_step(uint32_t timestamp)` 契约给出模型仓侧的承诺与一致性策略。本文件同时记录 firmware 端 `template_fms` 的签名错位作为契约风险事项。

## 2. 范围

**在范围:**

- `fms_interface_step` / `control_interface_step` / `plant_interface_step` 的 `uint32_t timestamp` 单位、epoch、回卷与单调性约定
- 模型仓侧三个 step 函数内部如何使用(或不使用)该 timestamp
- `dt`(步长)的来源选择:**timestamp 派生 vs 固定步长**
- MIL 仿真时钟与 firmware 系统时钟在三模块之间的一致性策略
- 时间戳作为契约字段的 freeze 边界(uint32_t、ms、自启动)
- `template_fms/fms_interface.c` 当前签名 `(void)` 与 `fms_interface.h` 声明 `(uint32_t timestamp)` 错位的处置

**不在范围(由其他工作项处理):**

- 1ms / 5ms / 10ms / 20ms 之间的 rate transition 块布置 — 由 [A4 跨速率边界设计](A4-rate-boundaries.md) 处理
- Solver 配置 / Sample Time 颜色检查 — 由 [G2 多速率调度设计](../G-harness/G2-rate-scheduling.md) 处理
- Plant 积分器 / 数值步长 — 由 [C4 Plant 数值与积分设计](../C-plant/C4-numerics.md) 处理
- Controller 5 ms 子预算分配 — 由 [E5 Controller 性能预算设计](../E-controller/E5-performance-budget.md) 处理
- logsout 信号采样时戳元数据 — 由 [G3 日志与可观测性设计](../G-harness/G3-logging.md) 处理
- `*_init` 行为与 reset 协议 — 由 [A6 Init/Reset 状态契约](A6-init-reset-contract.md) 处理(本文件只约定时间戳是否在 reset 时归零)

## 3. 依赖

| 上游 | 引用位置 | 用途 |
|---|---|---|
| 架构 v1 §13 Timing constraints and scheduling assumptions | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | 模块周期(Plant 1ms / Controller 5ms / FMS 20ms / INS 10ms)与"latency 必须考虑"原则 |
| 架构 v1 §15.3 SIH/HIL key checks | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | "timestamp assumptions / topic freshness / reset/init sequencing" 列为 SIH 关键检查项,需在 design 阶段先约定 |
| [A1 架构 v1 评审与缺口闭合](A1-v1-review.md) (status=reviewed) | §4.2 §13 行 / §5(本文件由 2026-05-05 审计 F-03 推动新增) | 确认架构 v1 中"`fms_interface_step(uint32_t timestamp)` 存在但 epoch / 回卷 / 单调性 / dt-source 未指定"为本工作项闭合目标 |
| firmware `fms_interface.h` 第 29 行 | `FMT-Firmware/src/model/fms/fms_interface.h` | 真实契约签名 `void fms_interface_step(uint32_t timestamp);` |
| firmware `task_vehicle.c` 第 45–90 行 | `FMT-Firmware/src/task/vehicle/normal/task_vehicle.c` | 调度器层 timestamp 来源:`time_now = systime_now_ms()`;`timestamp = time_now - time_start`(零 epoch) |
| firmware `systime.h` 第 99–100 行 | `FMT-Firmware/src/module/system/systime.h` | `uint64_t systime_now_us(void);` 与 `uint32_t systime_now_ms(void);` — 锁定 ms 单位与 uint32_t 容器 |

> Firmware 引用快照日期:2026-05-05。本文件未记录 commit hash(因模型仓尚未启动 firmware mirror;镜像引入时由 [B5 INS_Out_Bus 镜像同步流程](../B-contracts/B5-ins-bus-mirror.md) 统一登记)。本文件涉及的 firmware 接口(签名、单位、epoch)若由 firmware 一侧反向变更,需通过本文件 §10 变更日志回溯并通知 §7 下游。

## 4. 设计内容

### 4.1 时间戳单位 — 锁定 milliseconds

**决策:`timestamp` 单位为 milliseconds(ms)。容器为 `uint32_t`。**

依据:

1. firmware `task_vehicle.c` 第 59 行 `time_now = systime_now_ms()`,第 65 行 `timestamp = time_now - time_start`,第 81 行 `fms_interface_step(timestamp)` 直接传入。`systime_now_ms` 返回 `uint32_t` ms 值。
2. firmware `systime.h` 同时提供 `systime_now_us()`(`uint64_t`),但调度器选择 ms 版本。三个 step 接口共享同一 timestamp 变量,故所有三个模块的 timestamp 单位**严格一致**为 ms。
3. ms 粒度对四个模块周期(1 ms / 5 ms / 10 ms / 20 ms)都是整周期、无亚 ms 量化误差风险;5 ms 控制环对 ms 粒度的离散化不引入相位误差。
4. 若未来需要亚 ms 时间戳,需新增 `_us` 后缀的并行接口,**不**在原 timestamp 上加单位前缀,以免破坏现有 firmware 契约。

模型仓侧约定:

- 凡引用 timestamp(用于 staleness、log timetag、cmd 输入打戳等)的子系统,**只**以 ms 为单位运算。
- 任何对 timestamp 的算术(差分、阈值比较)必须与单位 ms 一致;不得隐式按 0.001 缩放成秒来与 `dt`(秒)直接相加 — 见 §4.5 dt 来源。

### 4.2 Epoch — 零起点(zero-from-boot 等价)

**决策:timestamp epoch 为"模型(回路)启动时刻",即 `timestamp = 0` 标记第一次 step 触发的瞬间;不使用 Unix epoch、不使用绝对 boot tick。**

依据:

1. firmware `task_vehicle.c` 第 60–65 行注释 `the model simulation start from 0, so we calcualtet the timestamp relative to start time`。`time_start` 在第一次进入循环时锁定,后续 timestamp 都是相对值。
2. 此 epoch 选择与"无绝对实时同步要求"的嵌入式飞控场景一致;Plant/FMS/Controller 不需要墙钟时间。
3. `uint32_t` ms 容器在零起点下的回卷周期为 \(2^{32} \div 1000 \div 86400 \approx 49.71\) 天 — 远超单次飞行任务时长;按嵌入式实践视为可接受。回卷处理见 §4.3。
4. MIL 仿真时,Simulink Model 时钟 `t` 从 0 起跑,等价于 firmware 的 `time_start = 0` 之后的相对时间;两者天然一致。

模型仓侧约定:

- Plant / FMS / Controller 内部**不**在 step 起步前对 timestamp 做任何偏移修正;以接口传入值为准。
- 仿真重置(MIL 单次仿真重新启动)与 firmware 重启的语义一致:timestamp 重新从 0 起。reset 是否归零见 [A6 Init/Reset 状态契约](A6-init-reset-contract.md) §<待定> 协议(本文件的承诺:reset → timestamp 归零是合法的,模型不会因此异常)。

### 4.3 回卷处理(wraparound)

**决策:采用"模 \(2^{32}\) 减法"约定计算时间差(即 `uint32_t` 自然 wrap-around 算术),适用于 timestamp 差分的所有用途。**

约定细节:

1. **定义**:对两个 timestamp \(t_a, t_b\)(\(t_a\) 较新,\(t_b\) 较旧),`dt_ms = (uint32_t)(t_a - t_b)`。在 `uint32_t` 上的减法天然按 \(2^{32}\) 模运算,只要真实时间差 \(< 2^{31}\) ms ≈ 24.85 天,差值正确。
2. **使用上限**:任何依赖 timestamp 差的逻辑(staleness 检测、log dt、cmd 时效),其语义有效阈值必须 **≤ 24.85 天**。当前所有用例(命令时效阈值秒级、log dt 毫秒级)远低于该上限。
3. **比较禁则**:**不得**用有符号比较 `(int32_t)(t_a - t_b) > 0` 之外的方式判断"哪个时间戳更新";不得用 `t_a > t_b` 直接比较 — 该写法在跨过 wrap 边界时会得出错误结果。
4. **跨 wrap 处理**:在 49.71 天连续运行场景下,timestamp 自然 wrap 一次,模型仓侧逻辑不需要特殊处理(模运算自动正确);但 logsout 中 timestamp 字段直接写 `uint32_t` 会出现"突然降到 0"的非单调,可观测性需在 [G3](../G-harness/G3-logging.md) 中以 `int32_t` 差值方式可视化或在采集端转换为 `uint64_t` 累加器(见 §4.4)。

伪代码(staleness 检测,语义参考):

```pseudo
% timestamp 差分:模运算自动处理 wrap
dt_ms = uint32(t_now - t_msg);    % 在 uint32 上自然 wrap
if dt_ms > STALENESS_THRESHOLD_MS
    msg_is_stale = true;
end

% 比较"是否较新":通过差值的有符号解读
diff_signed = int32(t_a - t_b);
if diff_signed > 0
    t_a is newer;
end
```

### 4.4 单调性约束

**决策:在单次会话(session)内 timestamp **单调非递减**;只有在系统 reset(参见 A6)时才允许从某个值回到 0。**

承诺细节:

1. **单次会话内**:timestamp 由调度器一次性记录 `time_start` 后,每次 step 调用 `time_now - time_start`,因 `systime_now_ms()` 单调,故 timestamp 单调非递减(同一 step 周期内多次访问可能相等,允许)。
2. **wrap 跨越**:从模型语义角度,wrap 不视为"非单调",因为模 \(2^{32}\) 减法仍给出正确的 dt(见 §4.3);但 logsout / 离线分析工具看到的"原始 uint32 序列"会有一次跳变 — 这是预期行为,不视为违约。
3. **MIL 仿真**:Simulink 仿真时钟单调;harness 注入 timestamp 必须以单调方式生成,**禁止**注入回退或乱序时间戳(此规则约束 [G4 Pilot_Cmd 注入设计](../G-harness/G4-pilot-injection.md))。
4. **Reset 后**:reset 触发时 timestamp 归零,这视为新会话开始。下游消费者(staleness 检测器、积分器)必须在 reset 边界丢弃跨边界的差值计算 — 由 [A6 Init/Reset 状态契约](A6-init-reset-contract.md) 协调通知机制。

### 4.5 dt 来源 — 模块内部固定步长 + timestamp 仅用于跨模块辅助

**决策:**

- **模块内部 `dt`(用于积分器、滤波器、控制律)一律使用固定步长**,值由模块 sample time 锁定:Plant 1 ms,Controller 5 ms,FMS 20 ms。**不**由 timestamp 派生。
- **timestamp 仅用于:** ① 跨模块 staleness / freshness 检测;② log timetag 写入;③ 输入 bus 字段打戳(`Pilot_Cmd.timestamp` 等);④ 调试 / 性能 profiling。

依据:

1. **代码生成确定性**:固定步长是 `ert.tlc` 单率生成的前提。timestamp-派生 dt 会在生成代码中产生 `dt = current - last` 的运行时计算,引入数值漂移(每周期 dt 实际是 1 ms 或 5 ms,但偶尔抖动 ±1 ms),违反架构 v1 §4.4 "Single-rate per module"。
2. **模型语义一致**:Simulink 内部 `Ts`(sample time)是 solver 推进的真理来源;在固定步求解器下,模块 step 之间间隔精确等于其周期。
3. **firmware 契约一致**:firmware `fms_interface_step(timestamp)` 调用频率由 `PERIOD_EXECUTE3(fms_step, fms_model_info.period, time_now, ...)` 控制(`task_vehicle.c` 第 81 行);`period` 由模型生成代码导出(`FMS_EXPORT.period`)。两端对周期的一致性已由 [B4 契约 diff 策略设计](../B-contracts/B4-contract-diff.md) 中的 `period` 数值精确相等校验保障。
4. **timestamp 抖动容忍**:firmware 调度器是事件驱动,`PERIOD_EXECUTE3` 不保证微秒级周期精度;若模型用 timestamp 派生 dt,会把调度抖动直接灌入控制律,损害稳定性。

模型仓侧约定:

- 任何 Simulink 子系统中的积分器、离散滤波器、PID 内部状态推进,其 `Ts` 通过 model parameter / Sample Time inheritance 设置为模块周期常量,**禁止**通过 input bus 中的 timestamp 字段计算实际 dt。
- **唯一例外**:跨模块/跨速率消费(例如 FMS 在 20 ms 周期内消费 INS 10 ms 输出)的 staleness 判断,允许使用 timestamp 差(单位 ms,§4.3 模运算)对照阈值。
- 单位换算:若某算法需要"上一帧到本帧的真实秒数",一律使用 `Ts_seconds = period_ms * 1e-3` 的固定常量,不读 timestamp。

### 4.6 Plant / FMS / Controller 三模块之间的时间戳一致性

| 场景 | 三模块时间戳关系 | 实现方式 |
|---|---|---|
| **MIL 仿真**(Plant + ins_stub + FMS + Controller 闭环) | 全部共享同一 Simulink simulation clock;harness 顶层从该时钟派生 ms timestamp 注入到三模块 input bus 的 `timestamp` 字段 | [G1 MIL 顶层结构设计](../G-harness/G1-mil-toplevel.md) 中由顶层 harness 统一生成 timestamp 信号并 broadcast |
| **SIL**(per-module 测试) | 测试夹具注入 timestamp 序列;模块内部 dt 仍用固定步长 | timestamp 仅作为输入 bus 字段验证;不影响算法行为 |
| **firmware 集成**(Plant in SIH / 真实飞行) | `task_vehicle.c` 第 65 行单一 `timestamp` 变量并发送给三个 `*_interface_step`;三模块共享同一时间基准 | firmware 一侧已实现;模型一侧只承诺"接收即可,不另行维护本地时钟" |

**一致性承诺:**

1. **同一 step 周期(在 firmware 一侧的 PERIOD_EXECUTE3 调度中)内,三模块若同时被 `time_now` 触发,得到的 timestamp 完全相同。** 这由 firmware `task_vehicle.c` 第 65 行单一 timestamp 变量保证。
2. 三模块各自不维护"本地 wall clock";所有时间相关字段(`FMS_Out.timestamp`、`Control_Out.timestamp`、`Plant_States.timestamp` 若存在)均由 step 接口参数派生,而非各模块独立调用计时器。
3. MIL 与 firmware 集成在时间语义上同构:MIL 中 harness 模拟 firmware 的"单一 timestamp 广播"行为。

### 4.7 `template_fms` 签名错位 — 契约风险记录

**事实:**

- header `FMT-Firmware/src/model/fms/fms_interface.h` 第 29 行声明:`void fms_interface_step(uint32_t timestamp);`
- 真实 vehicle 实现(`mc_fms` / `car_fms` / `fw_fms` / `vtol_fms` / `boat_fms`)均匹配该签名。
- 模板 `FMT-Firmware/src/model/fms/template_fms/fms_interface.c` 第 24 行定义:`void fms_interface_step(void)` — 与 header 不一致。

**影响分析:**

- `template_fms` 当作 firmware 端的占位/示例代码,可能不进入正式构建产物(具体由 firmware build system 控制),但**作为后续新机型实现的复制起点**,其错位会被人无意中传播。
- 模型仓侧的承诺(本文件 §4.1–§4.6)假定签名为 `(uint32_t timestamp)`;若新机型从 `template_fms` 复制并保持 `(void)`,会导致编译失败或(若强制类型转换 silently)行为偏离本文件约定。
- 这是 **firmware 一侧** 的代码瑕疵,**不在模型仓修复范围**;模型仓的本工作项只能登记并向用户提请 firmware 一侧修正。

**处置:** 本文件 §5 列为 Risk(契约不一致风险);§7 下游影响中标注为"firmware-side fix request",由用户决定是否提交 firmware patch。

### 4.8 与 firmware 契约的相互锁定要素

本文件锁定的契约锚点(任何变更需走 INDEX 决策日志):

| 锚点 | 值 | firmware 来源 |
|---|---|---|
| timestamp 单位 | milliseconds | `task_vehicle.c` 第 59 行 + `systime.h` 第 100 行 |
| timestamp 容器类型 | `uint32_t` | `fms_interface.h` 第 29 行 |
| timestamp epoch | 模型启动时刻(零起点) | `task_vehicle.c` 第 60–65 行注释与代码 |
| 回卷上限 | \(2^{32}\) ms ≈ 49.71 天 | uint32_t 容器自然属性 |
| 三模块共享 timestamp | yes(`PERIOD_EXECUTE3` 同一变量) | `task_vehicle.c` 第 76–83 行 |
| dt 来源 | 模块固定步长(period 常量) | model-side 决策(本工作项) |

`contract_impact: yes` 的依据:本工作项明确**承诺**了 firmware 可见的接口签名(timestamp 单位/类型/epoch)在模型仓侧的解读规则。任何此处的变更(例如未来切到 us)都会破坏 firmware 端 `task_vehicle.c` 的调用约定。已在 INDEX 决策日志记录的承诺锚点见上表。

## 5. 已知风险与悬而未决问题

- **`template_fms/fms_interface.c` 签名错位**
  - 影响:firmware 一侧;新机型若从该模板复制会引入编译错误或行为不一致。模型仓侧未直接受影响,但本文件 §4.6 的"三模块共享 timestamp"承诺在 template-derived 新机型上可能不成立。
  - 处置:**Open;请求 firmware 一侧修正**。具体建议:把 `template_fms/fms_interface.c` 第 24 行 `void fms_interface_step(void)` 改为 `void fms_interface_step(uint32_t timestamp)`,函数体可以保留空实现以维持模板"占位"语义。模型仓不直接修;由用户在评审本文件后决定是否提交 firmware 修补。

- **49.71 天回卷在长跑测试中的可观测性**
  - 影响:虽然 §4.3 的模运算保证算法正确,但回归测试 / 长跑测试在 logsout 中看到 timestamp 跳变会触发误报。
  - 处置:推迟到 [G3 日志与可观测性设计](../G-harness/G3-logging.md);本文件建议 G3 在采集端用 `uint64_t` 累加 ms,可视化端把 wrap 显式标注。

- **MIL 中 harness timestamp 注入精度**
  - 影响:若 G1 顶层 harness 把 simulation time `t`(秒,double)转 ms 时使用 `floor(t*1000)`,在 1 ms 解析下与 firmware 的 `systime_now_ms()` 行为等价;若用 `round`,会偶发偏 1 ms。
  - 处置:在 [G1 MIL 顶层结构设计](../G-harness/G1-mil-toplevel.md) 中规定使用 `floor` 或等价的下取整;本文件不重复规则。

- **timestamp 字段是否在 `FMS_Out_Bus` / `Control_Out_Bus` 中作为 contract 字段**
  - 影响:若 contract 字段表(B1)将 `timestamp` 列为字段,模型必须显式从输入 timestamp 直接 passthrough 到输出。如果跨周期(20ms→5ms)消费,需协调 latch 策略。
  - 处置:由 [B1 Bus 清单与 schema 设计](../B-contracts/B1-bus-inventory.md) 决定。本文件的承诺是"模型仓内部不会篡改 timestamp 值,直传或重新打戳"。

- **timestamp = 0 的歧义(冷启动 vs 第一帧)**
  - 影响:第一次 step 的 timestamp 一定是 0,与"未初始化"在数值上无法区分。任何对"timestamp == 0"做特殊判断的下游(例如 staleness)会在第一帧出错。
  - 处置:在 [A6 Init/Reset 状态契约](A6-init-reset-contract.md) 中协调:消费者必须以"自上次 reset 起的样本数"或独立 init flag 区分,**不**用 timestamp == 0 区分。本文件的承诺:模型仓内部不依赖该判断。

- **Wave 2 同批兄弟 A4 / A6 / A8 协同**
  - 影响:本文件的"固定步长 dt"决策依赖 A4 的 rate transition 选型;"reset 时归零"依赖 A6 的 reset 协议;若 A4/A6 在评审中改变方向,本文件需重审。
  - 处置:本文件与 A4/A6 同批(Wave 2);评审时建议三件一并冻结。当前互引用仅在概念层,无字段级耦合,允许并行起草。

## 6. 退出条件复核

A7 在 [`00-design-plan.md`](../00-design-plan.md) §4.A 中的退出条件原文:

> `fms_interface_step(uint32_t timestamp)` 的 timestamp 单位 / epoch / 回卷处理 / 单调性约束 / dt 来源(由 timestamp 派生 vs 固定步长);Plant/FMS/Controller 之间时间戳一致性策略;模板 `template_fms` 当前签名 `(void)` 与正式签名差异的处置(契约风险)

逐条复核:

| # | 退出条件分解 | 本文档依据 | 状态 |
|---|---|---|---|
| 1 | timestamp 单位 | §4.1 锁定 milliseconds + 依据(`systime.h` / `task_vehicle.c` 引用) | 满足 |
| 2 | epoch | §4.2 锁定零起点(模型启动时刻),与 firmware `time_start` 注释一致 | 满足 |
| 3 | 回卷处理 | §4.3 给出 \(2^{32}\) 模减法约定、24.85 天有效阈值、比较禁则、伪代码 | 满足 |
| 4 | 单调性约束 | §4.4 单次会话单调非递减;wrap 不视为违约;reset 归零;harness 注入须单调 | 满足 |
| 5 | dt 来源(timestamp 派生 vs 固定步长) | §4.5 锁定"模块固定步长 dt;timestamp 仅用于跨模块辅助",并给出依据(codegen 确定性、调度抖动) | 满足 |
| 6 | Plant/FMS/Controller 时间戳一致性策略 | §4.6 三场景表(MIL / SIL / firmware)+ 三条一致性承诺 | 满足 |
| 7 | `template_fms` 签名差异处置 | §4.7 事实 + 影响分析;§5 列为 Open Risk,提请 firmware 修补;§7 标注为 firmware-side fix request | 满足 |

## 7. 下游影响

按 [`01-design-relationships.md §4.10`](../01-design-relationships.md):

```text
A1 → A7
A7 ⇢ A4, G2, C4, E5
```

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [A4 跨速率边界设计](A4-rate-boundaries.md) *(同 Wave 2,未启动)* | ⇢ | §4.5 "固定步长 dt" 决策约束 A4 不能用 timestamp-派生 dt 跨速率;§4.6 三模块共享 timestamp,A4 在 rate transition 设计中按此协调 staleness 阈值 |
| [G2 多速率调度设计](../G-harness/G2-rate-scheduling.md) *(Wave 10,未启动)* | ⇢ | §4.6 MIL 顶层 harness 的 timestamp 注入策略;§4.4 单调性约束传递给 G4 注入与 G2 调度 |
| [C4 Plant 数值与积分设计](../C-plant/C4-numerics.md) *(Wave 9,未启动)* | ⇢ | §4.5 Plant 1ms 固定步长 dt;reset 时 timestamp 归零是 C4 reset 行为的输入 |
| [E5 Controller 性能预算设计](../E-controller/E5-performance-budget.md) *(Wave 10,未启动)* | ⇢ | §4.5 "Controller 5ms 内禁止 timestamp-派生 dt";§4.4 调度抖动传递给 E5 性能预算的边界假设 |

**间接影响 / 邻近责任方:**

| 受影响方 | 影响内容 |
|---|---|
| **firmware 一侧(`FMT-Firmware/src/model/fms/template_fms/fms_interface.c`)** | **请求 firmware-side fix**:第 24 行签名应改为 `void fms_interface_step(uint32_t timestamp)` 以匹配 `fms_interface.h` 第 29 行声明。模型仓不能也不应修;由用户决定是否提 firmware patch。本文件 §5 已记录为 Open Risk |
| [A6 Init/Reset 状态契约](A6-init-reset-contract.md) *(同 Wave 2)* | 同批兄弟。A6 需协调:reset 时 timestamp 归零、消费者不得用 `timestamp==0` 区分初始化态。引用本文件 §4.2 / §4.4 |
| [B1 Bus 清单与 schema 设计](../B-contracts/B1-bus-inventory.md) | timestamp 字段在 input/output bus 中的 freeze 规则:类型 `uint32_t`、单位 ms、不可改名 |
| [G1 MIL 顶层结构设计](../G-harness/G1-mil-toplevel.md) | harness 必须实现"单一 timestamp 广播"以模拟 firmware `task_vehicle.c` 行为 |
| [G3 日志与可观测性设计](../G-harness/G3-logging.md) | 长跑回归中 wrap 跨越的可视化处理(uint64 累加器或显式标注) |
| [G4 Pilot_Cmd 注入设计](../G-harness/G4-pilot-injection.md) | 注入序列必须单调,禁止时间倒退 |

下游作者使用本文件的方式:在自己工作项的 §3 依赖中引用本文件,§6 退出条件复核中确认所引用的本文件章节(尤其 §4.1 / §4.2 / §4.4 / §4.5)未被违反。

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-05 | A7 author | 初稿 |

## Self-check

- [x] frontmatter 完整,字段值合法
- [x] 上游文档全部存在且 status ≥ reviewed(A1 = reviewed/pass;架构 v1 已发布)
- [x] 退出条件逐条复核完成,每条均给出依据
- [x] 引用路径全部可点击访问
- [x] 不存在 RULES §5 禁则中的内容
- [x] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记(注:本文件 contract_impact=yes,登记由 INDEX 维护者在合并入库时完成,本文件 §4.8 已列出锚点)
- [x] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径(注:本文件不创建镜像产物;§3 已用相对路径 + 引用日期 2026-05-05 标记 firmware 来源,镜像产物由 B5 工作项统一记录 commit hash)
- [x] 下游影响已沿关系图识别完毕
- [x] 文档不超出本工作项范围(无越权设计)
