---
work_item: A6
title: Init/Reset 状态契约
upstream: ["架构 v1", "A1"]
contract_impact: yes
status: reviewed
authored_at: 2026-05-05
last_reviewed_at: 2026-05-05
reviewer_verdict: pass
---

# A6 Init/Reset 状态契约

## 1. 目的

为 FMT-Model-2025b 的 Plant / FMS / Controller 三个模型仓侧模块,在"启动初始化(init)"与"运行期复位(reset)"两类事件下,给出统一的**状态契约**:谁可以触发、何时触发、init 后哪些状态必须确定、reset 后各输出 bus 字段的稳态值类别(参数 / hardcoded / zero)、多模块之间在重启场景下的协调协议。本文件填补审计 [F-02](../_audit/2026-05-05-plan-audit.md) 指出的"无工作项拥有 init/reset 状态契约"缺口,并将架构 v1 §15.3 SIH key check 中的 "reset/init sequencing" 提升为可下游消费的设计契约。

## 2. 范围

**在范围:**

- `Plant_init` / `FMS_init` / `Controller_init` 在模型仓侧的**行为契约**(状态可决性、参数化形态、副作用边界)
- 运行期 reset 的**触发源、触发时机、生效范围**与跨模块协调协议
- reset 后各输出 bus(`FMS_Out_Bus` / `Control_Out_Bus` / `Plant_States_Bus` / `INS_Out_Bus` 消费侧默认)字段的**稳态值类别**(只规定"必须有定义"以及类别归属,不给数值)
- 初值来源分类法:**参数(B3)** / **hardcoded(模型内常量)** / **zero(默认零值)**
- MIL 仿真路径(`Plant_init` 实跑)与 firmware 路径(INS init 手写)在 init 行为上的**非对称性**及其约束
- 与 firmware 可见符号 `Plant_init` / `FMS_init` / `Controller_init` 的语义对齐(本文件 contract_impact=yes 的来源)

**不在范围(由其他工作项处理):**

- `*_init` 中各字段的**具体数值** — 由 [B3 Parameter schema 设计](../B-contracts/B3-parameter-schema.md) 处理
- 时间戳 / dt 来源 / epoch / 回卷 — 由 A7 时间约定处理(reset 触发时机若涉及时间语义,引用 A7)
- 跨速率边界处的 latch / hold / queue 在 reset 时的具体重置策略 — 由 [A4 跨速率边界设计](A4-rate-boundaries.md) 处理(本文件只规定"reset 必须使其回到 init 后状态")
- Plant 积分器的具体 reset 数值与 numerical re-initialization 细节 — 由 [C4 Plant 数值与积分设计](../C-plant/C4-numerics.md) 处理(本文件只规定"积分器 reset 行为必须显式定义")
- FMS 的 mode 状态机 reset 后的状态映射(进入哪个 mode)— 由 [D3 FMS Mode Manager 详细设计](../D-fms/D3-mode-manager.md) 处理(本文件只规定"reset 后 mode 必须为 deterministic 已知值")
- Controller 各环路 anti-windup 积分器与滤波器的 reset 数学约束 — 由 [E3 Controller 各环算法设计](../E-controller/E3-loops-algorithm.md) 处理
- INS 内部状态的 init/reset(firmware 手写,模型仓不建模)— 仅约束"消费侧对 INS_Out_Bus reset 期间字段如何处理"

## 3. 依赖

| 上游 | 引用位置 | 用途 |
|---|---|---|
| 架构 v1 §5.1 | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | `*_init` / `*_step` 是 firmware-visible 符号(§5.1.2 行 73–75);本文件契约必须与之兼容 |
| 架构 v1 §15.3 | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | SIH key check 含 "reset/init sequencing"(行 646);本文件把它提升为可消费的契约规范 |
| 架构 v1 §12.3 | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | INS 划归 firmware 原生;模型仓 init 不涵盖 INS 内部状态 |
| 架构 v1 §17 Risk 4 / OQ6 | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | INS 与模型仓侧 init 协议风险背景 |
| [A1 架构 v1 评审与缺口闭合](A1-v1-review.md) | reviewed | A1 §4.5.3 把契约 critical 集中在 B 区与 INS 镜像;本文件横切 init/reset 契约,与 A1 § 4.4 *(proposed)* 中的横切关注呼应 |
| [审计 F-02](../_audit/2026-05-05-plan-audit.md) | `00-design-plan §4` | 本工作项创建动因;明确指出 `*_init` firmware-visible 而 SIH key check 列入 reset/init sequencing |
| firmware 头文件 `FMT-Firmware/src/model/control/mc_controller/lib/Controller.h` | `extern void Controller_init(void);` / `extern void Controller_step(void);` | `Controller_init` 是无参 firmware-visible 符号;本契约的 init 参数化策略必须保留无参 codegen 入口 |
| firmware 头文件 `FMT-Firmware/src/model/fms/mc_fms/lib/FMS.h` | `extern void FMS_init(void);` / `extern void FMS_step(void);` | 同上,FMS init 入口无参 |
| firmware 头文件 `FMT-Firmware/src/model/plant/multicopter/lib/Plant.h` | `extern void Plant_init(void);` / `extern void Plant_step(void);` | 同上,Plant init 入口无参 |
| firmware 头文件 `FMT-Firmware/src/model/{control,fms,plant}/{control,fms,plant}_interface.h` | `*_interface_init(void)` 包装 | 模型仓生成的 `*_init` 由 firmware-side 的 `*_interface_init` 调用;本文件不涉及 interface 层细节,但保证 `*_init` 语义 |
| firmware 头文件 `FMT-Firmware/src/model/ins/ins_interface.h` | `INS_Status` / `INS_Flag` 位域 | INS 在 reset 期间通过 status/flag 位向消费者通告 readiness;本文件的 §4.5 INS 协调协议引用 |

注:firmware commit hash 须在本文件首次进入 `reviewed` 时由评审人或后续修订时记录(模板示例:`FMT-Firmware @ <40-char-sha>`)。当前 draft 暂以"路径相对引用 + 当前工作树状态"标记,待 INDEX 决策日志登记时补齐(见 §5)。

## 4. 设计内容

### 4.1 概念定义与术语

本节固定本契约用到的术语,以避免下游误读。

| 术语 | 定义 | 与 firmware 关系 |
|---|---|---|
| **init**(初始化) | 进程/模块生命周期内**仅一次**的状态建立动作。在 firmware 路径下由 `*_interface_init` → `*_init` 完成;在 MIL 路径下由仿真启动时调用 `Plant_init` / `FMS_init` / `Controller_init` 完成 | firmware-visible 符号 `FMS_init` / `Controller_init` / `Plant_init`(每个为 `void(void)`) |
| **reset**(运行期复位) | 模块运行期间,响应特定触发条件,把模块或其部分子状态恢复到一个**预定义稳态**的动作。**不**等同于重新调用 `*_init`(见 §4.2.4) | 在 firmware 中通常通过特定信号 / mode 切换 / param 重新链接触发;模型仓侧通过 `FMS_Out_Bus` 中的 `reset` 字段或 enable/disable 信号体现 |
| **稳态(post-init / post-reset)** | init 或 reset 完成后**第一次** `*_step` 执行前,模块所有可观察输出与所有内部存储状态都处于**已定义、可在文档中查到具体值**的状态 | 与 §4.4 "稳态值类别表"对应 |
| **firmware 路径** | Plant 不参与;`FMS_step` / `Controller_step` 由 firmware 调度器调用;INS 由 firmware 手写代码运行 | 架构 v1 §3 / §12.3 / §15.4 |
| **MIL 路径** | Plant / FMS / Controller / `ins_stub` 全部在 Simulink 中运行,顶层 harness 驱动 | 架构 v1 §15.1 |
| **deterministic** | 给定相同的初值来源(参数版本 / 模型版本)与外部输入序列,运行结果可复现到 byte-equal(用于 H3 回归基线) | 由 A7 时间契约 + 本文件 init/reset 契约共同保证 |

### 4.2 `*_init` 行为契约

#### 4.2.1 三个 init 函数的签名约束(契约影响 yes 的核心)

为兼容 firmware 既存 interface(架构 v1 §5.1.2),三个 init 必须保留**无参签名**:

```c
/* firmware-visible, MUST be preserved at codegen output */
extern void Plant_init(void);
extern void FMS_init(void);
extern void Controller_init(void);
```

**契约约束**:

1. 模型仓侧的设计**不得**在 init 入口引入参数(任何"配置"必须通过 PARAM struct 在 init 之前已被链接到模型工作空间或 firmware param 系统;PARAM 链接机制由 B3 设计)。
2. 任何尝试在 codegen 输出中改变 init 签名(例如生成 `Plant_init(struct PlantInitArgs*)`)被本契约**禁止**;契约 diff(B4 / I3)必须把这视作 hard fail。
3. INS 不在本契约范围内,但 `ins_interface_init(void)` 的存在(`FMT-Firmware/src/model/ins/ins_interface.h`)与本约束方向一致,纳入参考。

#### 4.2.2 init 的副作用边界

每个 `*_init` 的合法副作用集合:

| 副作用 | Plant_init | FMS_init | Controller_init |
|---|---|---|---|
| 初始化模块**内部状态存储**(stateflow chart 状态、unit-delay、积分器、滤波器、计数器)到 §4.4 稳态值 | 是 | 是 | 是 |
| 把模块**输出 bus** 写为 §4.4 稳态值 | 是 | 是 | 是 |
| 把模块**内部 bus / 总线总线**(若 leaf 与 shared 之间有内部 bus)写为稳态值 | 是 | 是 | 是 |
| 读取 PARAM struct(`PLANT_PARAM` / `FMS_PARAM` / `CONTROL_PARAM`,B3 定义) | 是(只读) | 是(只读) | 是(只读) |
| 申请动态内存 | **否**(架构 v1 §4.5) | **否** | **否** |
| 写日志 / 调用 RTOS API | **否**(模型代码不可调用 firmware API,违反契约 diff 规则) | **否** | **否** |
| 调度其他模块的 init / step | **否**(由 firmware interface 层负责) | **否** | **否** |
| 与 INS 通信 | **否**(INS firmware-owned) | **否** | **否** |
| 生成随机数 / 调用 RNG | **否**(确定性约束;若 ins_stub 需 RNG,由 H3 RNG 契约 + I 区脚本管理,不在 init 内) | **否** | **否** |

#### 4.2.3 init 必须使其确定的状态(分模块)

本节给出 init 完成后必须 deterministic 的**状态类别清单**;具体字段名由 B1 / B3 / 各模块结构设计文档(C2 / D2 / E2)交叉确定,本文件**不重述**。

**Plant**:
1. 刚体状态(位置 / 速度 / 姿态四元数 / 角速度)— 必须可决,具体类别(参数 vs hardcoded vs zero)见 §4.4
2. 电机状态、舵面/控制面状态(若有)— 可决
3. 大气 / 环境模型状态 — 可决
4. 传感器输出(`Plant_States_Bus` 的全部字段,`Extended_States_Bus` 的全部字段)— 可决
5. 数值积分器内部状态(详细数值约束见 C4)— 可决
6. 任何 unit-delay 块的初值 — 可决

**FMS**:
1. mode 当前状态(Mode Manager 的 stateflow chart;详见 D3)— 可决
2. mission 当前 waypoint / sequence index(详见 D5)— 可决
3. 安全监视器(failsafe latches、计数器)— 可决
4. command shaper 内部 rate / jerk filter 状态(详见 D4)— 可决
5. `FMS_Out_Bus` 所有字段(rate cmd / attitude cmd / cmd_mask / status / state / ext_state / ctrl_mode / wp / home / error 等,字段集见 B1)— 可决,且**默认安全态**(详见 §4.4.2)
6. `reset` 字段本身的初值(若 `FMS_Out_Bus` 含 reset 字段)— 见 §4.3

**Controller**:
1. 各环路(位置 / 速度 / 姿态 / 角速度)积分器与抗 windup 状态 — 可决(详见 E3)
2. 滤波器(LPF / D-term filter)状态 — 可决
3. 各环 enable / disable latch — 可决,默认 disable(直到 FMS 通过 cmd_mask 启用,详见 D6 / E1)
4. mixer / allocation 中间状态(若有)— 可决
5. `Control_Out_Bus` 所有字段 — 可决,且默认**电机零推力 / 舵面 trim**(详见 §4.4.2)

#### 4.2.4 init 与 reset 的关系

**init ≠ reset**。两者在本契约中是**不同事件**:

| 维度 | init | reset |
|---|---|---|
| 调用频率 | 仅一次(每次进程/仿真启动) | 多次,运行期 |
| 触发者 | firmware interface 层 / MIL harness | FMS(主) / 外部命令(受限,见 §4.3) |
| 涉及范围 | 整个模块状态 | 可能为部分子状态(局部 reset)或全部(全局 reset) |
| codegen 形态 | `*_init(void)` 函数 | `*_step` 内部分支(if-then-else 或 enable subsystem) |
| 状态终态 | §4.4 稳态值 | §4.4 稳态值(默认情况下与 init 后稳态**等价**;如有差异需在本文件 §4.4 显式列出) |

**关键约束:reset 后稳态默认与 init 后稳态相等**。如果某模块设计需要"reset 后状态 ≠ init 后状态"(例如 reset 时保留某些 mission 上下文),必须在本文件 §4.4 中**显式列出差异**并标注理由。本契约的默认值是"reset 即回到 init 后稳态"。

### 4.3 Reset 触发源、时机与权限矩阵

#### 4.3.1 触发源分类

定义五类合法 reset 触发源:

| 触发源代号 | 描述 | 谁能产生 | 模型仓侧体现 |
|---|---|---|---|
| **TS-FMS-OUT** | FMS 输出的 reset 命令(若 `FMS_Out_Bus` 含 reset 字段;字段名由 B1 锁定) | FMS 内部决策(safety / mode 切换) | `FMS_Out_Bus.reset` 信号置位 |
| **TS-EXT-CMD** | 外部(GCS / Pilot)显式 reset 命令 | GCS / Pilot 通过 FMS 输入 bus(`GCS_Cmd_Bus` / `Pilot_Cmd_Bus`) | FMS 把外部 reset 转译为 TS-FMS-OUT(集中式权威) |
| **TS-MODE-XCHG** | mode 切换隐含的局部 reset(例如从 manual 切到 position hold,某些环路积分器要清零) | FMS Mode Manager(详见 D3) | 由 FMS 决策,通过 cmd_mask 或 ctrl_mode 字段间接触发;Controller 在 step 内识别 ctrl_mode 跳变,做局部 reset |
| **TS-FAILSAFE** | failsafe 触发的安全 reset(geofence、低电、估计器失效等) | FMS Safety Monitor | 等价于一个特殊 mode 切换,走 TS-MODE-XCHG 路径,但额外置位 `FMS_Out_Bus.reset`(语义:全局 reset,优先级高于 mode 局部 reset) |
| **TS-MIL-HARNESS** | MIL 仿真仅 — harness 在场景脚本中显式调用 reset(用于场景拼接 / 故障注入恢复) | G1 顶层 harness(详见 G 区) | 通过仿真总线注入 reset 信号,**不**在 firmware 路径出现,契约 diff(B4 / I3)需保证此路径在 codegen 输出中被消除或编译开关隔离 |

**禁止的触发源**:
- Controller 内部决策触发 reset 自身或其他模块 — Controller 是受控方,不持有 reset 决策权
- Plant 内部决策触发 reset — Plant 不应有"自重置"行为(如有数值发散需要兜底,由 C4 用饱和而非 reset 解决)
- INS(firmware-owned)触发 Plant / FMS / Controller reset — INS 通过 `INS_Out_Bus` 中的 status/flag 字段(`INS_Status` / `INS_Flag` 位域,`FMT-Firmware/src/model/ins/ins_interface.h`)告知"未就绪",由 FMS 决策是否升级为 TS-FAILSAFE

#### 4.3.2 触发权限矩阵(谁可以使谁 reset)

| 触发模块 \ 受控模块 | Plant | FMS | Controller |
|---|---|---|---|
| **FMS** | 仅 MIL 路径(TS-MIL-HARNESS 经 FMS 转发);firmware 路径下 Plant 不在场 | 自身(Mode Manager 内部状态机) | 通过 TS-FMS-OUT / TS-MODE-XCHG / TS-FAILSAFE |
| **Controller** | 禁止 | 禁止 | 自身(局部环路积分器 reset,响应 cmd_mask 变化;详见 E3) |
| **Plant** | 自身(仅积分器在 step 内的内部数值健壮性兜底,不算 reset) | 禁止 | 禁止 |
| **harness(G1)** | 仅 MIL | 仅 MIL,通过 TS-MIL-HARNESS | 仅 MIL,通过 TS-MIL-HARNESS |
| **INS**(firmware-owned) | 不直接 reset,通过 status/flag 让 FMS 决策 | 不直接 reset | 不直接 reset |

**集中式权威**:运行期 reset 的**主权威是 FMS**。Controller 仅做对自身环路的、由 cmd_mask / ctrl_mode 派生的局部 reset。Plant 在闭环正常运行期不应被外部 reset(除 MIL harness 场景拼接需要)。

#### 4.3.3 触发时机与速率约束

- TS-FMS-OUT / TS-FAILSAFE:在 FMS 当前 step 周期内决定,在**下一 FMS step** 输出时生效;Controller 在收到 `FMS_Out_Bus` 后的**当前 Controller step** 内识别并执行局部 reset。
- TS-MODE-XCHG:与上同步;ctrl_mode 跳变在一个 Controller step 内被检测,reset 在该 step 内完成。
- TS-MIL-HARNESS:harness 控制时序,但不得在 step 函数运行中点触发(必须在 step boundary,即 sample hit 之间);由 G2 调度设计强制。
- **Reset 不在 sub-step 中点发生**。所有 reset 都对齐到模块的 sample boundary(Plant 1 ms / Controller 5 ms / FMS 50 Hz / INS 100 Hz 模型仓侧消费,详见架构 v1 §13)。

#### 4.3.4 触发的 guard / debounce 约束

- 同一触发源在连续两个 step 中重复置位 → 视为持续触发,模块仅在**首次** step 执行 reset 动作(幂等约束,避免 reset 风暴)。
- TS-FAILSAFE 不可被屏蔽;TS-EXT-CMD 在 FMS 中可被 mode 状态机 guard(例如 disarmed 状态下接受 reset,armed 状态下拒绝)。具体 guard 规则由 D1 / D3 决定,本文件不强制。
- 反方向:reset 信号撤回(置位→清零)**不**触发任何动作。reset 是**一次性事件**,不是"reset 状态"。

### 4.4 Reset 后 bus 字段稳态值类别契约

本节给出每个 firmware-visible 输出 bus 在**reset 完成后第一次 step 之前**,各字段必须取的"稳态值类别"。**类别**是契约,不是数值;数值在 B 区落地。

#### 4.4.1 类别定义

| 类别代号 | 含义 | 数值来源 | 修改成本 |
|---|---|---|---|
| **PARAM** | 来自 PARAM struct(`PLANT_PARAM` / `FMS_PARAM` / `CONTROL_PARAM`)的某字段 | runtime-tunable(B3 §"runtime-tunable 分类")或 compile-inlined(B3 §"compile-inlined 分类") | runtime 可调或重新生成(取决于 B3 分类) |
| **HARDCODED** | 模型内常量,不在 PARAM 中暴露 | 模型源代码 / Simulink 块的 constant 值 | 修改需重新生成 codegen |
| **ZERO** | 数值零 / 空向量 / 空四元数(具体形式见各 bus 字段类型) | 默认 init | 不可修改(契约) |
| **PRESERVED** | 不重置;reset 后保留 reset 前的值 | 上一周期值 | 仅适用于 reset 不应清除的"上下文"字段(如 mission progress 在某些 reset 类型下保留);使用此类别需附理由 |

**默认类别**:任何字段如果本文件未指定,默认为 **ZERO**。

#### 4.4.2 各输出 bus 的类别契约表

> 下表只给出**字段类别**而非字段名清单。完整字段清单由 B1 锁定,本表用类别覆盖该清单的全部字段。

**`Plant_States_Bus`**(Plant 输出,MIL 路径下消费者为 ins_stub / harness;firmware 路径不存在)

| 字段类别 | 类别 | 备注 |
|---|---|---|
| 刚体位置 / 速度 / 姿态 / 角速度 | PARAM(初值由 `States_Init_Bus` 提供) | `States_Init_Bus` 见架构 v1 §5.1 / §8.1 |
| 电机/舵面状态 | ZERO 或 PARAM(选择由 C2/C3 决定) | 多旋翼 leaf 通常 ZERO;固定翼可能需要 trim PARAM |
| 数值积分器内部状态 | 由 C4 决定,但必须落入 PARAM / HARDCODED / ZERO 之一 | C4 不得引入 PRESERVED 类别 |

**`Extended_States_Bus`**:全字段 ZERO 默认,允许 C2 显式声明 PARAM 例外。

**`Environment_Info_Bus`**(Plant 输出之一):全字段 PARAM(标准大气、风场参数等都是配置量)。

**`FMS_Out_Bus`**(FMS 输出,firmware-visible 关键)

| 字段类别 | 类别 | 备注 |
|---|---|---|
| `cmd_mask` | HARDCODED(全 0,即"无环路启用") | 安全态:Controller 看到 cmd_mask=0 应输出零推力 |
| `ctrl_mode` | HARDCODED(对应 disarmed/idle 的 enum 值;具体值由 B2 锁定) | enum 值由 B2 定;本契约只规定"必须是 deterministic 的 disarmed/idle 等价值" |
| `mode` | HARDCODED(disarmed/manual/idle 中由 D3 选定) | 对齐 D3 mode 集合 |
| `status` / `state` / `ext_state` | HARDCODED(disarmed 等价 enum) | 同上 |
| 速率 / 姿态 / 速度 / 加速度 命令字段 | ZERO | 即使 cmd_mask=0,字段值也必须是 0(避免下游误读) |
| `actuator_cmd` passthrough | ZERO | |
| `throttle_cmd` | ZERO | |
| `wp` / `home` / `error` 等 | PARAM 或 ZERO,由 D2 决定 | wp 默认 ZERO;home 视设计可为 PARAM |
| `reset` 字段(若存在) | ZERO(一次性事件位,稳态为 0) | |

**`Control_Out_Bus`**(Controller 输出,firmware-visible 关键)

| 字段类别 | 类别 | 备注 |
|---|---|---|
| 电机推力 / PWM 命令 | HARDCODED(零推力 / disarmed PWM) | 多旋翼安全态;具体数值由 E4 决定 |
| 舵面命令 | HARDCODED(trim 中位 / 由 E4 决定) | 仅多旋翼以外机型相关,本切片不强制 |
| Controller 内部状态镜像(若 Control_Out_Bus 含调试字段) | ZERO | |

**`INS_Out_Bus`**(消费侧默认;模型仓**不**生成 INS,只规定 FMS / Controller 在 INS 未就绪时的契约)

| 字段类别 | 类别 | 备注 |
|---|---|---|
| status / flag(`INS_Status` / `INS_Flag` 位域) | INS firmware-owned;模型仓不约束其值,但**必须**消费 `ready` / `*_valid` 位作为 gate | 见 §4.5 |
| 各 navigation 字段 | 模型仓不约束;消费侧必须在 `flag.bit.<x>_valid == 0` 时使用 fallback(由 F3 INS 消费规则定义) | F3 必须显式约定 fallback 行为 |

#### 4.4.3 稳态值的 self-consistency 约束

- **Plant 输出与 ins_stub 输出在 init 后必须 self-consistent**:`ins_stub` 在 init 后产生的 `INS_Out_Bus` 应反映 Plant 的初值(例如 Plant 初始 hover,则 `INS_Out_Bus.position` 应为 hover 位置)。该一致性由 F1 / F2 实现保证,但本契约要求"必须存在一致性"。
- **FMS_Out_Bus 与 Controller_Out_Bus 在 init 后必须 self-consistent**:cmd_mask=0 下 Controller 不应主动输出非零控制量。
- self-consistency 不依赖 init 顺序(三个 init 的调用顺序不应影响最终稳态;仅影响中间过渡期,过渡期长度 ≤ 1 step)。

### 4.5 多模块协调协议

#### 4.5.1 init 调用顺序契约

- **firmware 路径**:由 firmware interface 层(`fms_interface_init` / `control_interface_init` / `plant_interface_init` / `ins_interface_init`)依次调用对应的 `*_init`。模型仓侧**不**约束顺序,但要求"任意顺序下,稳态最终一致"(§4.4.3)。
- **MIL 路径**:由 G1 顶层 harness 调用,推荐顺序为 `Plant_init` → `ins_stub_init` → `FMS_init` → `Controller_init`(理由:Plant 提供初始 truth,`ins_stub` 据此产出 `INS_Out_Bus`,FMS 据此产出 `FMS_Out_Bus`,Controller 最后启动)。但本契约**不强制**该顺序,因为稳态自洽约束已使顺序不影响终态。

#### 4.5.2 reset 协调协议

定义两类 reset:**全局 reset(global)** 与 **局部 reset(local)**。

| 类型 | 触发源 | 协调规则 |
|---|---|---|
| **global reset** | TS-FMS-OUT(`reset` 字段=1)/ TS-FAILSAFE / TS-MIL-HARNESS | FMS 在当前 step 完成自身全部子状态 reset;同 step 输出 `FMS_Out_Bus` 含 `reset=1`、cmd_mask=0、ctrl_mode=disarmed-equiv;Controller 在下一 step 看到 `reset=1` 后,本周期内 reset 全部环路状态、输出 §4.4.2 中 Control_Out_Bus 的稳态值;**Plant 在 firmware 路径下不参与**;在 MIL 路径下,Plant reset 由 harness 显式触发(不通过 FMS_Out_Bus.reset 联动) |
| **local reset** | TS-MODE-XCHG(ctrl_mode 跳变 / cmd_mask 部分 bit 翻转) | FMS 不重置自身;Controller 仅 reset 受影响的环路(由 D6 cmd_mask 语义决定);Plant 不参与 |

**关键约束**:
1. **Global reset 不联动 Plant reset**(在 firmware 路径下 Plant 不存在;在 MIL 路径下 Plant 物理状态不应被 FMS 决策清除,否则违反"FMS 不掌握 Plant"的角色边界,架构 v1 §7)。如果 MIL 场景需要联动重置 Plant,由 harness 显式发起 TS-MIL-HARNESS。
2. **INS 协调**:global reset 不能"reset INS"(INS firmware-owned)。INS 通过自己的 status/flag 位汇报状态;FMS 在 reset 完成后必须等待 `INS_Flag.bit.ready==1` 才离开 disarmed/idle 状态(具体守卫由 D3 实现)。这是 firmware 路径下 INS init 与模型仓 init 的**异步协调**。
3. **Reset 优先级**:同一 step 内 TS-FAILSAFE > TS-FMS-OUT(权威) > TS-EXT-CMD(经 FMS) > TS-MODE-XCHG(局部);TS-MIL-HARNESS 在 MIL 路径下与 TS-FAILSAFE 同级。
4. **跨速率边界**:reset 在 sample boundary 对齐(§4.3.3),跨速率 transition 块(latch / hold,A4 定义)必须在 reset 当周期把内部缓存清回稳态。本文件不规定具体 reset 机制,仅约束"必须重置"。

#### 4.5.3 INS 协调协议(firmware 路径专项)

INS 是 firmware-owned(架构 v1 §3 / §12.3),其 init 由 firmware 手写代码完成,**模型仓侧无法直接控制**。本契约规定**模型仓侧消费者**(FMS / Controller)的契约:

1. FMS 在 `FMS_init` 后输出的 `FMS_Out_Bus` 必须处于 disarmed/idle 安全态(§4.4.2),与 INS 是否就绪无关。
2. FMS 在每个 step 内,首先检查 `INS_Flag.bit.ready`;若为 0,保持 disarmed/idle 状态(具体守卫由 D3 实现)。
3. Controller 不直接判断 INS readiness;Controller 看 cmd_mask;FMS 通过 cmd_mask 间接传达 INS 状态。
4. 在 SIH/HIL(架构 v1 §15.3),"reset/init sequencing" 的检查项就是验证此协议在 firmware 调度下成立。

#### 4.5.4 协议时序图(文本表示)

**global reset 时序(firmware 路径)**:

```text
step k (FMS):
    failsafe / external cmd 触发
    -> FMS reset 内部状态
    -> 输出 FMS_Out_Bus { reset=1, cmd_mask=0, ctrl_mode=disarmed-equiv, ... }

step k or k+1 (Controller, 取决于调度对齐):
    读到 FMS_Out_Bus.reset == 1
    -> Controller reset 全部环路状态
    -> 输出 Control_Out_Bus { 零推力 / trim, ... }

step k+m (FMS, 后续):
    输出 FMS_Out_Bus.reset = 0(一次性事件已发出)
    继续监视 INS_Flag.ready 决定是否离开 disarmed/idle

INS:
    不参与 reset 决策;由 firmware 调度独立运行
```

**local reset 时序(mode 切换,firmware 路径)**:

```text
step k (FMS):
    mode 切换决策(D3 触发)
    -> 输出 FMS_Out_Bus { cmd_mask 部分位翻转, ctrl_mode 改变, reset=0, ... }

step k or k+1 (Controller):
    检测 ctrl_mode 跳变 / cmd_mask 变化
    -> 仅 reset 受影响环路的积分器与 anti-windup 状态
    -> 其他环路 / 状态保留

Plant: 不参与
INS:   不参与
```

### 4.6 初值来源分类与命名义务

#### 4.6.1 三类初值来源

| 类别 | 适用 | 引用 |
|---|---|---|
| **PARAM** | 任何"配置量",尤其是物理几何 / 控制增益 / mode 阈值 / hover 初始位置 | B3 必须为本 PARAM 字段标 **runtime-tunable** 或 **compile-inlined** |
| **HARDCODED** | 安全态默认(disarmed enum 数值、cmd_mask=0、零推力 PWM 等)与契约级常量 | 出现在模型 constant 块或代码生成的 `#define`/`const` 中;契约 diff(B4 / I3)需保护其数值不漂移 |
| **ZERO** | 任何无明确选择的字段(默认) | 不在 PARAM,不在 HARDCODED,即 ZERO |

**禁止的反模式**:
- "在 init 中由 RNG 决定的初值" — 违反 deterministic;若需 RNG,由 ins_stub / 故障注入(F1 / H4)管理,**不**进入 init。
- "在 init 中读取外部文件 / 环境变量" — 违反 codegen 可移植性;所有配置必须经 PARAM 路径(B3)。
- "在 init 中调用其他模块的 step" — 跨模块同步由 firmware interface 层负责,模型仓 init 不得跨模块。

#### 4.6.2 命名义务(传递给 A2)

为了让 init/reset 行为可追溯,本文件约束 A2 命名规则补充如下条目(由 A2 落地具体规则,本文件**不**重复定义):

- 模型工作空间 / Simulink Data Dictionary 中**安全态默认值**的常量需带固定前缀(如 `SAFE_DEFAULT_*`)。
- PARAM 中作为"初值"使用的字段需在 B3 中标注 `init_use=true`(B3 schema 字段)。

(以上为对 A2 / B3 的下游建议,A2 / B3 在自己的退出条件中决定是否采纳。)

### 4.7 MIL stub vs firmware reality 的非对称性

#### 4.7.1 非对称点列表

| 非对称项 | MIL 路径 | firmware 路径 | 处置 |
|---|---|---|---|
| Plant 是否存在 | 存在,`Plant_init` 实跑 | 不存在 | MIL 必须验证 Plant_init 后稳态(§4.4.2 Plant_States_Bus 类别表) |
| INS 是否被建模 | `ins_stub` 模型仓侧 | INS 手写 firmware | 消费侧契约一致(§4.5.3);ins_stub 必须模拟 INS_Status / INS_Flag 位域,使 FMS 同样 gate(由 F1 实现) |
| reset 触发可由 harness 注入 | TS-MIL-HARNESS 合法 | 不存在 | codegen 输出中 TS-MIL-HARNESS 路径必须被消除 / 编译开关隔离;契约 diff(B4 / I3)在符号存在性检查时不应漏失任何 firmware 路径符号 |
| RNG / 噪声 / 偏置 | ins_stub / 故障注入(H4)有 | INS 内部确定性运行(其内部噪声模型由 firmware 决定,模型仓不可见) | RNG seed 管理由 H3 / I 区脚本约定,不进入 init/reset 契约本身 |
| 故障注入恢复 | TS-MIL-HARNESS 可触发 reset 测试恢复 | 通过 firmware failsafe 路径(等价 TS-FAILSAFE) | H4 故障目录要求每条故障必须给"恢复路径"对应到本契约的 reset 类别 |

#### 4.7.2 对称性保留约束

在两条路径的**交集**(FMS / Controller)上,init/reset 行为必须**完全等价**:

- 任何 reset 后稳态在 MIL 与 firmware 路径下产生 byte-equal 的 `FMS_Out_Bus` / `Control_Out_Bus`(给定相同 PARAM 与相同输入序列)。
- TS-MIL-HARNESS 路径不得在 firmware 输出中引入额外 step 分支(否则破坏 SIL 等价性,架构 v1 §15.2)。

此对称性保留是 H3 回归基线策略的隐式输入(H3 在自己的退出条件中决定是否引用本节)。

## 5. 已知风险与悬而未决问题

- **firmware commit hash 暂缺**
  - 影响:RULES §5 要求镜像自 firmware 的契约记录 commit hash + 文件相对路径。本文件 §3 仅给出文件相对路径,commit hash 留待评审或后续修订时补充。
  - 处置:open;评审通过前由 reviewer 或 author 补全,纳入 INDEX 决策日志同步登记(本工作项 contract_impact=yes,登记是 DoD 的强制项,见 RULES §8)。

- **`FMS_Out_Bus.reset` 字段是否真的存在于当前契约,需 B1 落实**
  - 影响:§4.3.1 / §4.4.2 / §4.5.4 假设 `FMS_Out_Bus` 含 `reset` 位字段;A1 §4.2 §5.1.5 提到"reset"作为 FMS_Out_Bus 字段之一,但未确认。若 B1 锁定后发现不存在,本契约的 global reset 信道需改用 cmd_mask + ctrl_mode 派生(可行,但流程更隐式)。
  - 处置:open,与 B1 协同确认;若需修改,本文件 §4.3 / §4.5 走 RULES §10 变更日志,B3 / D6 / E1 受影响。

- **Plant_init 在 firmware 路径下"不存在" vs "存在但未被调用"的歧义**
  - 影响:架构 v1 §5.1 行 67 列出 `plant_interface_init`,意味着 firmware 也可能编译/链接 Plant(用于 SIH/HIL)。本契约 §4.7.1 用"firmware 路径不存在 Plant"是简化表述;实际 firmware 可能在 SIH 编译时仍包含 Plant_init。
  - 处置:在 SIH/HIL 设计阶段明确(本设计阶段不涉及 SIH;架构 v1 §15.3 已提及"reset/init sequencing 是 SIH key check");本文件 §4.7 标注此简化,留待 SIH 实施时再行精化。

- **TS-MIL-HARNESS 路径在 codegen 中的消除机制未指定**
  - 影响:依赖 I4 codegen 配置 + 编译开关(`#ifdef MIL_BUILD` 之类)实现;本文件不重述,但 I4 必须在自己的退出条件覆盖此点。
  - 处置:在与 I4 协同时验证;若 I4 不覆盖,本文件 §5 留 open。

- **PRESERVED 类别在 reset 后保留 mission 上下文的合法性**
  - 影响:§4.4.1 引入 PRESERVED 类别,但 §4.4.2 默认未使用。若 D5 多旋翼 leaf 设计决定"global reset 后保留 home / current_wp 字段",必须在 D5 中显式声明并回填本文件 §4.4.2 的 wp / home 行。
  - 处置:open;由 D5 根据具体多旋翼 mission 需求裁决。

- **reset 在 ins_stub 中的语义未定义**
  - 影响:§4.5.2 仅讨论 Plant / FMS / Controller;MIL 路径下 ins_stub 是否随 global reset 重置内部噪声状态(尤其 RNG seed)未规定。
  - 处置:由 F1 ins_stub 功能设计决定;本文件不强制,F1 必须给定 stub reset 行为(可选:重新播种 / 保持原 seed / 切换到下一 seed)。

- **本工作项是审计新增项,A1 文档未对 A6 给出预派指派内容**
  - 影响:本文件依赖于 A1 §4.5.3 与 A1 §5 的横切关注点说明,但 A1 中没有把缺口逐条 "Closed by: A6" 标记;本文件采取主动声明的方式覆盖架构 v1 §15.3 (reset/init sequencing) 与 §5.1 (`*_init` firmware-visible 符号)的相关条目。
  - 处置:open;建议在 A1 文档进入下一次修订时,补一行"§15.3 reset/init → A6";由 INDEX 决策日志登记。

## 6. 退出条件复核

A6 在 [`00-design-plan.md`](../00-design-plan.md) §4.A 中的退出条件原文:

> `*_init` 行为定义、reset 触发源与时机、初值来源(参数 vs hardcoded)、reset 后的 bus 字段稳态值、Plant/FMS/Controller 在重启场景下的协议

逐条复核:

| # | 退出条件子项 | 本文档依据 | 状态 |
|---|---|---|---|
| 1 | `*_init` 行为定义 | §4.2 全节(签名约束 §4.2.1、副作用边界 §4.2.2、必须确定的状态 §4.2.3、init vs reset §4.2.4) | 满足 |
| 2 | reset 触发源 | §4.3.1 五类触发源表 + §4.3.2 权限矩阵 + §4.3.4 guard | 满足 |
| 3 | reset 时机 | §4.3.3 时机与速率约束(对齐 sample boundary、跨 step 行为)+ §4.5.4 时序图 | 满足 |
| 4 | 初值来源(参数 vs hardcoded) | §4.4.1 类别定义 (PARAM / HARDCODED / ZERO / PRESERVED) + §4.6 来源分类与命名义务 | 满足 |
| 5 | reset 后的 bus 字段稳态值 | §4.4.2 各 bus 类别契约表(Plant_States_Bus / Extended_States_Bus / Environment_Info_Bus / FMS_Out_Bus / Control_Out_Bus / INS_Out_Bus 消费侧)+ §4.4.3 self-consistency | 满足(契约形式;具体数值由 B3 落地,符合 A1 § 4.4 与 RULES §4 单一来源原则) |
| 6 | Plant / FMS / Controller 在重启场景下的协议 | §4.5 多模块协调协议(全节);§4.5.1 init 顺序 + §4.5.2 global vs local reset + §4.5.3 INS 协调 + §4.5.4 时序图 | 满足 |
| 7 | (隐含)与 firmware 可见契约一致 | §3 引用 firmware 头文件(Controller.h / FMS.h / Plant.h);§4.2.1 强制保留无参签名;§4.7 处理 MIL vs firmware 非对称性 | 满足(contract_impact=yes 已声明) |
| 8 | (隐含)缺口对应于审计 F-02 与架构 v1 §15.3 | §1 目的明确引用 F-02;§3 引用架构 v1 §15.3;§5 留 open 项指向 SIH 阶段 | 满足 |

## 7. 下游影响

按 [`01-design-relationships.md §4.10`](../01-design-relationships.md):

```text
A1 → A6
A6 ⇢ B3, C1, C4, D1, E1
```

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [B3 Parameter schema 设计](../B-contracts/B3-parameter-schema.md) *(未启动)* | ⇢ | §4.4 类别契约要求 B3 标注每个 PARAM 字段是否被 init 使用(`init_use=true`);§4.6.1 PARAM 类别的 runtime-tunable / compile-inlined 进一步分类(B3 已有此退出条件)是本契约的承接;§4.6.2 命名义务(`SAFE_DEFAULT_*` 前缀提议)由 A2 / B3 协商落地 |
| [C1 Plant 功能设计](../C-plant/C1-plant-functional.md) *(未启动)* | ⇢ | §4.2.3 Plant 必须确定的状态清单;§4.4.2 Plant_States_Bus / Extended_States_Bus / Environment_Info_Bus 类别契约;§4.7.1 MIL vs firmware 非对称性中 Plant 部分 |
| [C4 Plant 数值与积分设计](../C-plant/C4-numerics.md) *(未启动)* | ⇢ | §4.2.3 Plant.6(unit-delay / 积分器初值可决);§4.4.2 数值积分器内部状态类别约束(C4 不得引入 PRESERVED);§4.5.2 reset 不能用 Plant 自重置代替数值健壮性 |
| [D1 FMS 功能设计](../D-fms/D1-fms-functional.md) *(未启动)* | ⇢ | §4.3 FMS 是 reset 主权威;§4.4.2 FMS_Out_Bus 各字段类别;§4.5.3 INS 协调协议(FMS 必须 gate `INS_Flag.bit.ready`);§4.5.2 global vs local reset;D3 的 mode 状态机 reset 后的 mode(本文件不指定具体值,只要求 deterministic) |
| [E1 Controller 功能设计](../E-controller/E1-controller-functional.md) *(未启动,Wave 8)* | ⇢ | §4.2.3 Controller 必须确定的状态清单;§4.4.2 Control_Out_Bus 类别;§4.5.2 local reset(cmd_mask 派生)与 global reset 在 Controller 侧的实现要点 |

A6 还**间接影响**(按拓扑):

- D3 / D4(Mode Manager / Command Shaper):reset 后的 mode 与 shaper 状态;由 D2 / D3 / D4 落地
- D5 / E4(多旋翼 leaf):leaf 对 §4.4.2 表的 PARAM 类别字段提供具体数值
- E3(Controller 各环算法):各环 anti-windup 与滤波器 reset 行为(本文件 §4.2.3.Controller.1-3)
- E5(Controller 性能预算):reset 路径不应突破 5 ms 预算(本文件 §4.3.3 sample-boundary 约束有助于此)
- F1(ins_stub 功能):§4.5.3 + §4.7.1 (RNG / reset 在 stub 中的语义)
- F3(INS 消费规则):§4.5.3 由 F3 落地具体 fallback 字段值
- G1 / G2(harness):TS-MIL-HARNESS 路径接入点;§4.3.1 / §4.7.1
- H4(故障注入目录):每类故障的"恢复路径" → §4.3.1 reset 类别映射
- I3 / I4(契约 diff / codegen 配置):§4.2.1 init 签名保护;§4.7.2 对称性保留

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-05 | A6 author | 初稿(响应 2026-05-05 plan-audit F-02) |

## Self-check

- [x] frontmatter 完整,字段值合法
- [x] 上游文档全部存在且 status ≥ reviewed(架构 v1 已发布;A1 status=reviewed,见 [INDEX](../INDEX.md) 已完成清单)
- [x] 退出条件逐条复核完成,每条均给出依据(§6 表给出 6 项明确条件 + 2 项隐含条件)
- [x] 引用路径全部可点击访问
- [x] 不存在 RULES §5 禁则中的内容(无 .slx 截图、无可执行 .m、无 firmware 实现复述、无 PR/branch 名;本文件未重复 bus/enum 字段表,而是给"类别"契约;FMT-Firmware 契约已记录文件相对路径,§5 标注 commit hash 待补)
- [x] 触及 firmware 契约者(contract_impact=yes)将由 orchestrator 在 INDEX 决策日志登记(任务 brief 已声明此责任;本文件作为 author 不直接编辑 INDEX,符合 fmt-design-author skill 规则)
- [ ] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径(**部分满足**:文件相对路径已在 §3 给出;commit hash 待 reviewer / orchestrator 在 INDEX 登记同步补齐,见 §5 第 1 项 open)
- [x] 下游影响已沿关系图识别完毕(§7 表覆盖 B3 / C1 / C4 / D1 / E1 五条直接出边 + 间接下游枚举)
- [x] 文档不超出本工作项范围(无越权设计;具体 PARAM 数值留 B3、积分器细节留 C4、mode 状态留 D3、环路细节留 E3,本文件仅给契约)
