---
work_item: G4
title: Pilot_Cmd 注入设计
upstream: ["架构 v1", "A2", "A6", "A7", "B1", "B2", "D1", "D4", "G1"]
contract_impact: no
status: draft
authored_at: 2026-05-10
last_reviewed_at:
reviewer_verdict: none
---

# G4 Pilot_Cmd 注入设计

## 1. 目的

为 Phase 2 MIL 闭环验证提供**脚本驱动的 Pilot_Cmd 时间序列注入机制**:固化 (a) 注入到 `Pilot_Cmd_Bus` 的字段集合(per B1 §4.2.1)与 G1 顶层 Pilot_Cmd_Source 子系统的对接点、(b) 主用 CSV 时间序列文件格式、(c) 高层场景描述 DSL(`scenario.yaml`)、(d) stick 整形与 G4 输出之间的"原始 vs 整形后"分工、(e) 时间戳与 reset/init 行为、(f) Phase 5+ 实时 RC 直通的接口预声明。本文件**不**重定义 `Pilot_Cmd_Bus` 字段、**不**重定义 `PilotMode` 数值、**不**实现 stick 整形、**不**实现 sim runner;以上分别由 B1 / B2 / D4 / I5 拥有。

## 2. 范围

**在范围:**

- G4 注入产物到 `Pilot_Cmd_Bus`(per B1 §4.2.1)字段的映射(注入点)
- 主用 CSV 时间序列文件格式(列、采样率、命名约定、单位语义)
- 可选高层场景 DSL `scenario.yaml`(事件清单、duration、动作枚举)
- DSL → CSV 的预处理职责归属(I5 接口约定,不实现)
- 注入时间戳与 A7 的对齐策略(harness clock 单一来源)
- Init / reset 行为(预 CSV 起始默认 = "no input")
- Phase 5+ 实时 RC / 摇杆直通的接口预声明
- 与 D4 stick 整形的"原始 vs 整形后"分工说明

**不在范围(由其他工作项处理):**

- `Pilot_Cmd_Bus` 字段级 schema / 字节布局 — 由 [B1 §4.2.1](../B-contracts/B1-bus-inventory.md) 拥有;G4 仅 echo
- `PilotMode` enum 名 / 数值 — 由 [B2 §4.4.4](../B-contracts/B2-enum-inventory.md) 拥有;G4 仅 echo
- Stick deadband / expo / 单位换算 — 由 [D4 §4.3.1 / 起 Block (2)](../D-fms/D4-command-shaper.md) 处理;G4 输出**未整形**的 raw stick
- Pilot_Cmd_Source 子系统在 harness 中的物理位置 / 与 FMS 的连线 — 由 [G1 MIL 顶层结构设计](G1-mil-toplevel.md)(Wave 10 sibling)拥有
- Pilot_Cmd_Bus → FMS 跨速率边界(RB-06)的策略 / latency — 由 [A4 §4.4 RB-06](../A-architecture/A4-rate-boundaries.md) 拥有;G2 echo
- timestamp 单位 / epoch / 回卷 / 单调性 — 由 [A7](../A-architecture/A7-time-conventions.md) 拥有;G4 仅消费
- 仿真运行器 CLI / 报告 / 批跑 — 由 [I5 sim runner 脚本设计](../I-tooling/I5-sim-runner.md) 拥有
- GCS / Auto / Mission 命令源注入 — 不在 G4 范围(本文件仅注入 `Pilot_Cmd_Bus`;其他源由 H1 / H4 在场景层组合,G4 不做)
- logsout 信号采样规则 — 由 [G3 日志与可观测性设计](G3-logging.md)(Wave 10 sibling)拥有

## 3. 依赖

| 上游 | 引用位置 | 用途 |
|---|---|---|
| 架构 v1 §16 Smallest executable slice | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | "arm → takeoff/hold → manual or position hold → land/disarm" 闭环 MIL 场景 — 本文件场景示例的依据 |
| 架构 v1 §15.1 MIL | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | MIL 命令注入只需 scripted Pilot;不依赖真实 RC |
| [A2 命名与目录约定](../A-architecture/A2-naming-conventions.md) (status: reviewed) | §4.10 单位与坐标系 / R-10.x | CSV 列命名(`stick_roll` 等)与 `_n01` 单位前缀对齐;场景目录命名遵循 R-1.x |
| [A6 Init/Reset 状态契约](../A-architecture/A6-init-reset-contract.md) (status: reviewed) | §4.4 类别表 / §4.5 reset 协调 | "预 CSV 起始 = no input" 对应 HARDCODED 类别;G4 reset 行为约束 |
| [A7 时间与时间戳约定](../A-architecture/A7-time-conventions.md) (status: reviewed) | §4.1 ms / §4.2 epoch / §4.4 单调 | CSV 中 `timestamp_ms` 的单位 / epoch / 单调要求,与 firmware `task_vehicle.c` `time_now - time_start` 等价 |
| [B1 Bus 清单](../B-contracts/B1-bus-inventory.md) (status: reviewed) | §4.2.1 `Pilot_Cmd_Bus` | 10 字段 schema(`roll/pitch/yaw/throttle_stick_n01` / `mode_switch` / `arm_switch` / `kill_switch` / `valid` / `aux_channel[8]` / `timestamp`)— 注入字段全集 |
| [B2 Enum 清单](../B-contracts/B2-enum-inventory.md) (status: reviewed) | §4.4.4 `PilotMode` | `PMODE_NONE/MANUAL/STABILIZE/ALTHOLD/POSHOLD/MISSION/RTL/LAND/OFFBOARD` 9 成员 — DSL `mode_change.target` 取值集 |
| [D1 FMS 功能设计](../D-fms/D1-fms-functional.md) (status: reviewed) | §4.2.1 PILOT 行 / §4.2.2 / §4.3.1 | 注入意义("PILOT 是 mandatory 命令源,M-02..M-05/M-12 主导,M-06..M-11 作为 pilot intervention");G4 注入序列即 D1 §4.2.1 PILOT 源的输入侧 |
| [D4 FMS Command Shaper 详细设计](../D-fms/D4-command-shaper.md) (status: reviewed) | §4.3.1 / Block (2) Stick Conditioning | G4 输出 raw stick;deadband / expo / 单位换算 全部在 D4 Block (2) 进行;G4 不参与 |
| [G1 MIL 顶层结构设计](G1-mil-toplevel.md)(同 Wave 10 batch — known-loose,co-seal 风格引用) | Pilot_Cmd_Source 子系统接入点 | G4 注入 = G1 子系统拓扑中 Pilot_Cmd_Source 块的内容;详见 §4.1 |
| [01-design-relationships.md §4.7](../01-design-relationships.md) | `G1 → G4` 边 | G4 在 G1 之后但同 Wave 10;按 RULES §6 / playbook §3.4 视为 co-seal 风格(known-loose,见 §3 batch 注) |

**Co-seal note**:本文件与 G1 / G2 / G3 / E5 同属 Wave 10。orchestrator wave plan 把 G1 与 G4 放入同一 Wave 10 并行(known-loose:`G1 → G4` 强前置在关系图 §4.7 存在),作者按 co-seal batch 兄弟规则起草:G4 假定 G1 的 Pilot_Cmd_Source 子系统**存在**且接受 `Pilot_Cmd_Bus` 输入,但不依赖 G1 的具体子系统命名 / 块内拓扑;G1 在评审时复核本文件 §4.1 注入点的兼容性。

**Firmware 引用快照**:本文件不直接镜像 firmware 代码;`Pilot_Cmd_Bus` schema 与 `PilotMode` 数值经由 B1 / B2 间接锚定到 `FMT-Firmware/src/model/fms/<vehicle>/lib/FMS_types.h @ <pending hash>`(per B1 §4.2.1 / B2 §4.4.4)。

## 4. 设计内容

### 4.1 Pilot_Cmd_Bus 注入点(per B1 §4.2.1)

G4 产出**一条** `Pilot_Cmd_Bus` 时间序列,在 G1 顶层 harness 的 Pilot_Cmd_Source 子系统中物化为 bus 信号。注入字段**全集 echo B1 §4.2.1**;G4 不增删字段:

| Order | 字段(per B1) | 类型(per B1) | G4 输入(CSV 列) | 注入语义 | 备注 |
|---:|---|---|---|---|---|
| 1 | `roll_stick_n01` | `single` ∈ [-1, +1] | `stick_roll` | raw pilot roll stick(D4 整形前) | per A2 R-10.x `_n01` 单位 |
| 2 | `pitch_stick_n01` | `single` ∈ [-1, +1] | `stick_pitch` | raw pilot pitch stick | |
| 3 | `yaw_stick_n01` | `single` ∈ [-1, +1] | `stick_yaw` | raw pilot yaw stick | |
| 4 | `throttle_stick_n01` | `single` ∈ [0, +1] | `stick_throttle` | raw pilot throttle stick(单边量程) | range 与其他三轴不同(per B1 第 4 行) |
| 5 | `mode_switch` | `uint8` (PilotMode per B2 §4.4.4) | `pilot_mode` | RC mode 拨杆当前位置(枚举 name) | CSV 用 enum **name 字符串**,转换在 §4.2.3 |
| 6 | `arm_switch` | `boolean` | `arm_request` | arm 拨杆位置(0/1) | rising edge → FMS 仲裁 arm |
| 7 | `kill_switch` | `boolean` | `kill_request` | kill 拨杆位置(0/1) | optional;TR-10 触发 |
| 8 | `valid` | `boolean` | (隐式 = 1 在场景跨度内;§4.6) | RC 链路有效位 | 默认 1;RC link loss 场景由 `valid=0` 显式注入 |
| 9 | `aux_channel` | `single[8]` ∈ [-1, +1] | `aux1`..`aux8` | RC 辅助通道(图传 / 灯 / 钩等) | optional 列;缺省 0;长度 8(per B1 占位) |
| 10 | `timestamp` | `uint32` ms | `timestamp_ms` | per A7;harness clock 派生 | §4.5 |

**注入行为契约**:

1. G4 把 CSV 的每一行(或 DSL 解析出的每个 sample)在对应 `timestamp_ms` 时刻**整体**写入 `Pilot_Cmd_Bus`(同一 sample 的所有字段在同 step 同时生效,避免字段间时序错位)。
2. G4 输出的 stick 值是 **raw [-1, +1]**(throttle 为 [0, 1]):任何 deadband / expo / unit-mapping **由 D4 Block (2) 处理**(§4.4)。
3. G4 不做 INS / arm 状态机条件:G4 只**回放**注入序列;arm/kill/mode 是否实际生效由 D1 §4.2.2 仲裁 + D3 mode 状态机决定。
4. `Pilot_Cmd_Bus` 进入 FMS 经 RB-06(per A4 §4.2);G4 不参与 RB-06 块部署(由 G2 §4.3 拥有)。

**G4 不注入的字段**:无。`Pilot_Cmd_Bus` 全 10 字段都由 G4 提供(CSV 默认值或 DSL 派生)。

### 4.2 主用时间序列格式:CSV

#### 4.2.1 格式选型

**主用格式 = CSV(UTF-8,逗号分隔,行尾 LF)**。

依据:

1. **可读 / 可脚本生成**:CSV 是 plain text;场景作者可手写、Python/Bash 脚本生成、Excel 编辑。
2. **可被 Simulink / MATLAB 直接读取**:`readtable` / `From Spreadsheet` 块原生支持;无需自定义 reader。
3. **可 git diff**:与 Phase 2 设计先行 + 配置即代码原则一致(架构 v1 §11.2)。
4. **回归基线兼容**:H3 回归基线策略(`docs/design/H-verification/H3-regression-baseline.md` 待启动)对 CSV 的 binary-equality 比较成本低。

**可选辅助格式 = `MAT.gz`(gzip 压缩 MATLAB v7.3 .mat)**,仅在以下情形使用:

- 单场景行数 > 100 000(典型 50 Hz × 2000 s = 100 000 行,Phase 2 不会超),或
- 包含 nontrivial 浮点精度(高分辨率扫频 stick 输入)需保留 IEEE-754 完整 mantissa,文本 CSV 因截断引入数值误差。

Phase 2 闭环验证场景(架构 v1 §16:arm → takeoff → hold → land,典型时长 ≤ 60 s)**全部使用 CSV**;`MAT.gz` 接口预声明但 Phase 2 不部署。

#### 4.2.2 列定义

CSV 第一行 = header,列名严格如下:

| 列名 | 类型 | 取值约束 | 必填 | 注入到 Pilot_Cmd_Bus 字段 |
|---|---|---|---|---|
| `timestamp_ms` | `uint32` | 单调非递减,首行 = 0(per §4.5) | yes | `timestamp` |
| `pilot_mode` | string | `PMODE_NONE / PMODE_MANUAL / PMODE_STABILIZE / PMODE_ALTHOLD / PMODE_POSHOLD / PMODE_MISSION / PMODE_RTL / PMODE_LAND / PMODE_OFFBOARD`(per B2 §4.4.4) | yes | `mode_switch`(name → enum 数值,§4.2.3) |
| `arm_request` | int | `0` 或 `1` | yes | `arm_switch` |
| `kill_request` | int | `0` 或 `1` | yes | `kill_switch` |
| `stick_roll` | float | [-1.0, +1.0](per A2 R-10.x `_n01`) | yes | `roll_stick_n01` |
| `stick_pitch` | float | [-1.0, +1.0] | yes | `pitch_stick_n01` |
| `stick_yaw` | float | [-1.0, +1.0] | yes | `yaw_stick_n01` |
| `stick_throttle` | float | [0.0, +1.0]单边 | yes | `throttle_stick_n01` |
| `aux1`..`aux8` | float | [-1.0, +1.0],缺省 0 | optional | `aux_channel[0..7]` |
| `valid` | int | `0` 或 `1`,缺省 1 | optional | `valid` |

**未列出的列名**:CSV reader 必须报错(防止拼写错误悄悄丢字段)。

**列顺序**:`timestamp_ms` 必须在第一列;其他列顺序自由(reader 按 header 名解析,不按 position)。

#### 4.2.3 PilotMode name → enum 数值转换

CSV 中 `pilot_mode` 列写 enum **name 字符串**(如 `PMODE_STABILIZE`),不写裸数值。理由:

1. **可读性**:scenario 作者直接看 CSV 能理解意图,无需查 B2 表。
2. **drift 兼容**:B2 §4.4.4 锁定 name → 数值映射;若 firmware 数值变化,B5 mirror 流程同步;G4 CSV 无需重写。
3. **lint 友好**:CSV reader 校验 name 在 §4.2.2 9 成员 enumeration 内(预防拼写错误);裸数值无法做该校验。

转换在 G4 注入侧(harness)完成:CSV 解析 → 字符串查 B2 表 → 写入 `Pilot_Cmd_Bus.mode_switch`(`uint8`)。Phase 2 实施时,该映射表是 I2 镜像脚本产出物之一(per B5);G4 不维护独立映射,只**消费**。

#### 4.2.4 采样率

| 项 | 值 | 依据 |
|---|---:|---|
| 推荐采样率 | **50 Hz**(20 ms 每行) | 与 FMS 周期(20 ms / 50 Hz,架构 v1 §13)同频:每个 FMS step 都有一个 fresh sample,无需 G4 端做超采样 |
| 最大采样率 | 100 Hz(10 ms 每行) | 对齐 firmware 真实 RC 链路典型 50–100 Hz(per B1 §4.2.1 nominal rate) |
| 最小采样率 | 1 Hz(1000 ms 每行) | 长 hold 段(场景 5 s 静止 hold)允许稀疏 |

**G4 注入端在 sample 之间的行为 = hold-last(zero-order hold)**:任意时刻 t,`Pilot_Cmd_Bus` 字段值 = 最近一行 `timestamp_ms ≤ t` 的字段值。该 ZOH 行为与 RB-06(per A4 §4.4)的 single-element queue + Latch 语义自然兼容:CSV 自身的 ZOH 在 G4 内,RB-06 在 harness layer。

**插值禁止**:G4 不做线性 / 高阶插值,理由:

1. RC pilot 信号在硬件层就是 PWM 离散刷新,line interpolation 是合成产物,无物理意义。
2. 任何 stick 平滑应在 D4 Block (2) 用 first-order LP 实现(per D4 §4.3.1 line "可选 first-order LP for stick smoothing,D5 leaf"),不能在 G4 端做。
3. `pilot_mode` / `arm_request` / `kill_request` 是离散信号,插值无定义。

#### 4.2.5 文件命名与路径

per A2 R-1.x 命名规则:

- 存放路径:`model/harness/scenarios/<scenario_name>/pilot_cmd.csv`
- `<scenario_name>` 使用 lower_snake_case(per A2 R-1.x 文件名规则),例如:
  - `model/harness/scenarios/hover_takeoff_test/pilot_cmd.csv`
  - `model/harness/scenarios/manual_acro_demo/pilot_cmd.csv`
  - `model/harness/scenarios/rc_link_loss_failsafe/pilot_cmd.csv`
- 同目录下若存在 `scenario.yaml`(§4.3),CSV 由 DSL 预处理生成,在生成时自动覆写,不应手工与 YAML 混编。

#### 4.2.6 示例

最小示例(arm-takeoff-hold-land 序列,简化):

```csv
timestamp_ms,pilot_mode,arm_request,kill_request,stick_roll,stick_pitch,stick_yaw,stick_throttle
0,PMODE_NONE,0,0,0,0,0,0
1000,PMODE_NONE,1,0,0,0,0,0.0
2000,PMODE_STABILIZE,1,0,0,0,0,0.5
3000,PMODE_POSHOLD,1,0,0,0,0,0.5
8000,PMODE_POSHOLD,1,0,0.3,0,0,0.5
9000,PMODE_POSHOLD,1,0,0,0,0,0.5
25000,PMODE_LAND,1,0,0,0,0,0.5
30000,PMODE_NONE,0,0,0,0,0,0
```

行解读:

- `t=0` ms:仿真启动,pilot 未输入,disarm。
- `t=1000` ms:arm 拨杆置位,但仍 `PMODE_NONE`(等待 INS gate)。
- `t=2000` ms:切换 STABILIZE,油门到 hover 中位 0.5。
- `t=3000` ms:切换 POSHOLD(D3 守卫由 D5 决定;G4 只负责注入,不预判 mode 是否被接受)。
- `t=8000`–`9000` ms:1 秒 roll 杆 +0.3(stick step 测试)。
- `t=25000` ms:切 LAND,自动降落。
- `t=30000` ms:disarm,仿真即将结束。

完整闭环场景示例与 §4.3 DSL 等价输入对照见 §4.3.5。

### 4.3 场景描述 DSL(`scenario.yaml`)

#### 4.3.1 引入动机

CSV 适合**精细帧级**控制,但不适合**重复性场景批量构造**。例如"50 个不同 takeoff 高度的 hover 测试"在 CSV 中需要 50 个文件,每个 200+ 行;在 DSL 中是一段 5 行配置 + 一个高度数组。DSL 是 CSV 的高层 generator,**可选**;Phase 2 闭环最小切片(架构 v1 §16,5 个场景内)允许直接手写 CSV。

DSL → CSV 的转换由 [I5 sim runner](../I-tooling/I5-sim-runner.md) 在批跑前预处理(I5 自身设计在 Wave 12;G4 仅声明接口契约)。

#### 4.3.2 文件位置与格式

- 路径:`model/harness/scenarios/<scenario_name>/scenario.yaml`
- 格式:YAML 1.2(UTF-8)
- 与同目录 `pilot_cmd.csv` 的关系:**单一来源**。如果两者并存,I5 必须报错;典型工作流:作者写 `scenario.yaml`,I5 生成同目录 `pilot_cmd.csv`(标记 `# auto-generated from scenario.yaml at <ISO timestamp>`)。

#### 4.3.3 顶层 schema

```yaml
name: <string>                # 场景名;必须与目录名一致
description: <string>         # 可选;一句话场景目的
duration_s: <float>           # 仿真总时长,秒
sample_rate_hz: <int>         # 生成 CSV 的采样率,默认 50;§4.2.4
seed: <int>                   # 可选;若场景含随机扰动(本文件不定义,留 H4 故障注入)
events:
  - t: <float>                # 触发时刻,秒
    action: <enum>            # 见 §4.3.4 动作枚举
    <action-specific-params>
```

`events` 列表按 `t` 升序排列;I5 解析时校验单调。

#### 4.3.4 动作枚举(Phase 2 集合)

每条 event 的 `action` 字段必须从下表选取。每个动作展开成一段 CSV 行序列(由 I5 实现,本文件只规定语义)。

| action | 必填参数 | 可选参数 | 展开成 CSV 的语义 |
|---|---|---|---|
| `arm` | — | — | 在 `t` 时刻把 `arm_request` 从 0 → 1;后续保持 1 直至 `disarm` |
| `disarm` | — | — | 在 `t` 时刻 `arm_request` 1 → 0;清零所有 stick |
| `kill` | — | — | 在 `t` 时刻 `kill_request` 0 → 1(immediate;TR-10 触发) |
| `unkill` | — | — | 在 `t` 时刻 `kill_request` 1 → 0 |
| `mode_change` | `target: <PilotMode name>` | — | 在 `t` 时刻 `pilot_mode` 切到 `target`(B2 §4.4.4 9 成员之一) |
| `throttle_set` | `target: <float [0,1]>` | — | 在 `t` 时刻 `stick_throttle` 立即设为 `target`,持续 hold |
| `throttle_up` | `target: <float [0,1]>`, `duration: <float s>` | — | 在 `[t, t+duration]` 期间 `stick_throttle` 从当前值 ramp 到 `target`(线性) |
| `throttle_down` | `target: <float [0,1]>`, `duration: <float s>` | — | 同上,反向 |
| `stick_step` | `axis: <roll/pitch/yaw>`, `value: <float [-1,1]>`, `duration: <float s>` | — | 在 `[t, t+duration]` 期间 `stick_<axis>` = `value`,期满恢复 0 |
| `stick_pulse` | `axis`, `value`, `pulse_width: <float s>` | — | 单个矩形脉冲;`pulse_width` 后归 0 |
| `aux_set` | `channel: <1..8>`, `value: <float>` | — | 在 `t` 时刻 `aux<channel>` 设为 `value`,hold |
| `link_loss` | `duration: <float s>` | — | 在 `[t, t+duration]` 期间 `valid` = 0(RC link loss 注入,失败安全测试用) |
| `hold` | `duration: <float s>` | — | 在 `[t, t+duration]` 期间所有字段 hold-last(显式语义,等价于不写任何 event) |

**Phase 2 决策**:动作枚举不允许 `acro_pattern` / `mission_replay` / `figure_eight` 等高阶模板;此类场景在 Phase 2 直接写 CSV。Phase 4 / 5 视需要扩展(由 H1 / H4 触发,届时通过 INDEX 决策日志登记)。

#### 4.3.5 完整 DSL 示例(对应 §4.2.6 的 hover_takeoff_test)

```yaml
name: hover_takeoff_test
description: arm → STABILIZE → POSHOLD → 1s roll step → LAND → disarm
duration_s: 30
sample_rate_hz: 50

events:
  - t: 0.0    action: hold          duration: 1.0
  - t: 1.0    action: arm
  - t: 2.0    action: mode_change   target: PMODE_STABILIZE
  - t: 2.0    action: throttle_set  target: 0.5
  - t: 3.0    action: mode_change   target: PMODE_POSHOLD
  - t: 8.0    action: stick_step    axis: roll  value: 0.3  duration: 1.0
  - t: 25.0   action: mode_change   target: PMODE_LAND
  - t: 30.0   action: disarm
```

I5 把上述 YAML 展开为 §4.2.6 CSV(50 Hz,1500 行)。语义严格等价。

#### 4.3.6 DSL ↔ CSV 等价性约束

为 H3 回归基线起见,以下不变量必须由 I5 保证:

1. **可重放性**:同一 YAML 在同一 I5 版本下展开必须产出 byte-identical CSV(deterministic generation)。
2. **CSV 反向不可生成 YAML**:CSV 是低层产物,YAML 是高层意图;I5 不实现 CSV → YAML 反向工具(信息丢失)。
3. **作者优先**:用户可手工编辑 YAML 或 CSV 二选一,**禁止同目录共存**(§4.3.2)。

### 4.4 与 D4 stick 整形的分工:G4 输出 raw,D4 整形

per [D1 §4.3.1](../D-fms/D1-fms-functional.md) + [D4 §4.3.1 / Block (2)](../D-fms/D4-command-shaper.md),pilot stick 在被 FMS 当作 setpoint 之前要经过 stick conditioning(死区 / expo / 单位换算 / mode-条件 max 限值映射)。该处理**全部在 D4 Block (2) Stick Conditioning 内完成**,G4 不参与。

明确分工(场景作者参考):

| 数据形态 | 范围 | 拥有方 |
|---|---|---|
| **Raw stick**(G4 输出 / `Pilot_Cmd_Bus` 字段值) | `[-1, +1]`(throttle [0, 1]),无死区 / 无 expo / 无单位映射 | G4(本文件)+ B1 §4.2.1 schema 锁定 |
| **Conditioned stick**(D4 整形后 / Block (2) 输出) | 死区清零、expo 曲线后、映射到物理单位(rad / rad·s⁻¹ / m·s⁻¹ / 0..1) | D4 §4.3.1 / Block (2) |
| **Setpoint**(D4 整形完毕进入 cascade)| 物理单位 + jerk/rate-limited | D4 §4.4 模式行 + B1 `FMS_Out_Bus` |

**对场景作者的含义**:

- 想让 vehicle 静止 hover →`stick_throttle = 0.5`(假定 D5 leaf 把 0.5 映射到 hover 油门 per `manual_thr_hover_n01` FMS_PARAM.41);**不是** `stick_throttle = <hover_thrust_N>`。
- 想触发 hover 上 0.3 弧度倾角 → `stick_roll = 1.0`(全杆量),期望 D4 把 1.0 映射到 `tilt_lim_rad`(FMS_PARAM.38)= 0.3 rad;**不是** `stick_roll = 0.3`。
- 想测试 deadband 行为(stick = 0.04 应被 deadzone 抹平)→ G4 直接注入 `stick_roll = 0.04`;D4 deadband(默认 PARAM .40 = 0.05)清零;FMS 看到 0。

**G4 范围终点**:G4 注入 raw 值,**不**核对 deadband / expo / max 限值是否合理 — 那是 D4 行为,在 H1 验证场景中通过 logsout 比较 D4 输出验收。

### 4.5 timestamp 策略(per A7)

#### 4.5.1 单一时钟来源

per [A7 §4.6](../A-architecture/A7-time-conventions.md),MIL 仿真中 harness 顶层从 Simulink simulation clock 派生 ms timestamp 并 broadcast 到三模块 input bus 的 `timestamp` 字段。G4 注入的 `Pilot_Cmd_Bus.timestamp` 必须**与 harness 主时钟一致**:

- G4 不维护独立时钟。CSV 中的 `timestamp_ms` 是**场景在仿真启动后的相对时刻**(per A7 §4.2 epoch = 模型启动时刻 = 0)。
- harness 把 simulation time(秒,double)转 ms 时使用 `floor(t * 1000)`(per A7 §5 third bullet);G4 CSV 中 `timestamp_ms` 是已经按 ms 整数化的值。
- G4 / harness 不允许用 `round`(可能偶发 ±1 ms 偏移)。

#### 4.5.2 单调性

per [A7 §4.4](../A-architecture/A7-time-conventions.md) 单次会话内 timestamp 单调非递减:

- CSV 中 `timestamp_ms` 必须严格单调非递减(允许相邻两行相等,代表"同一时刻多个字段的复合 sample"是不必要的,但语法上允许)。
- I5 解析 CSV 时校验该不变量;违反 → 场景拒绝运行。
- DSL 展开时 I5 必须保证生成的 CSV 单调(`events[].t` 已要求升序,§4.3.3)。

#### 4.5.3 单调性下限:CSV 首行 timestamp = 0

- per A7 §4.2 epoch 零起点,CSV 第一行 `timestamp_ms` 必须 = 0(不是 1, 20 或负值)。
- 若场景设计者希望"前 5 秒空闲再 arm",写 `t=0` 行 mode=NONE / 全零 + `t=5000` 行的实际事件;**不**省略 `t=0` 行。这与 §4.6 init 行为协同。

#### 4.5.4 timestamp 不参与 dt 计算

per [A7 §4.5](../A-architecture/A7-time-conventions.md),模块内部 dt **由固定步长决定**,timestamp 仅用于跨模块陈旧检测 / log / profiling。G4 注入的 timestamp 字段:

- 在 RB-06 latch(per A4 §4.4)中由 FMS 用作 staleness 检测的对照点(per D1 §4.2.3)。
- **不**被 D4 / D3 拿去算 dt(D4 用 FMS 周期 20 ms 常量)。

### 4.6 Init / reset 行为

#### 4.6.1 Pre-CSV-start:HARDCODED zero

per [A6 §4.4](../A-architecture/A6-init-reset-contract.md) 类别表 + §4.6 的 G4 注入责任:

- 在 `harness boot` 后到 CSV 第一行(`timestamp_ms = 0`)生效之前的瞬间(物理上是同一 step,但 Simulink 块 init 顺序可能在 t=0 step 内的某个微秒级时刻早于 CSV 解析),`Pilot_Cmd_Bus` 必须处于"no input"安全态:

| 字段 | 安全态值 | 类别(per A6) |
|---|---|---|
| `roll_stick_n01` | 0 | HARDCODED |
| `pitch_stick_n01` | 0 | HARDCODED |
| `yaw_stick_n01` | 0 | HARDCODED |
| `throttle_stick_n01` | 0 | HARDCODED |
| `mode_switch` | `PMODE_NONE`(数值 0,per B2 §4.4.4) | HARDCODED |
| `arm_switch` | 0 (false) | HARDCODED |
| `kill_switch` | 0 (false) | HARDCODED |
| `valid` | 0 (false) — RC link 未 ready | HARDCODED |
| `aux_channel[0..7]` | 0 | HARDCODED |
| `timestamp` | 0 | HARDCODED |

注意 `valid = 0` 在 pre-CSV-start 时刻:这与 D1 §4.2.3 一致 — 任何 source 的 `valid==0` 在仲裁中**视为不存在**,FMS 不会因此误读 stick / mode。**一旦 CSV 第一行(t=0)开始注入,`valid` 默认转 1**(除非场景显式注入 `valid=0`,如 `link_loss` event)。

#### 4.6.2 Reset(运行期复位)行为

per [A6 §4.5.2](../A-architecture/A6-init-reset-contract.md) global reset:

- **TS-MIL-HARNESS 触发的全局 reset**(场景拼接 / 故障注入恢复,A6 §4.3.1):G4 必须把 CSV 注入指针**回退到第一行**,等同于"重新开始播放";`Pilot_Cmd_Bus` 同步置为 §4.6.1 安全态。
- **TS-FMS-OUT / TS-FAILSAFE 触发的全局 reset**(由 FMS 决策,A6 §4.5.2):G4 **不参与**自动回退 — pilot CSV 是外部输入源,不响应内部 reset 信号。这是 A6 §4.5.2 关键约束 1 ("global reset 不联动 Plant reset",对应延伸到外部输入源 Pilot/GCS/Auto/Mission)的合理推论:外部源不被 FMS 内部决策"清空"。
- 因此 G4 reset 的**唯一触发**是 TS-MIL-HARNESS;Phase 2 闭环最小切片(架构 v1 §16)中无场景拼接,实际不会触发。

#### 4.6.3 CSV 结尾行为

到达 CSV 最后一行 `timestamp_ms = T_last` 之后:

- 若 `t > T_last`:`Pilot_Cmd_Bus` **hold-last**(最后一行的字段值持续输出),直至仿真结束(由 G1 / I5 设置的 `stop_time`)或 TS-MIL-HARNESS reset。
- 不自动回到 §4.6.1 安全态。理由:hold-last 与 ZOH 语义一致(§4.2.4),且 Phase 2 场景习惯在最后一行写 disarm-equivalent 行(`pilot_mode=PMODE_NONE, arm_request=0, all sticks=0`),hold-last 该行天然安全。
- 场景作者**应**在 CSV 末尾写一行 disarm 行(如 §4.2.6 的 `t=30000` 行),作为防御性收尾。

### 4.7 Phase 5+ Variant:实时 RC / 摇杆直通

#### 4.7.1 接口预声明

Phase 5+(per 架构 v1 §18 Phase 5)若希望接入真实 RC 接收机或 USB 摇杆做实时驱动,G4 子系统应暴露**变体接入点**(具体 Variant 块由 G1 / A5 决定);本文件预声明接口契约,**不**实现:

| Phase 2 默认 variant | "Scripted CSV" | 本文件 §4.1–§4.6 |
|---|---|---|
| Phase 5+ optional variant | "Live passthrough" | 接受 RC PWM / 摇杆 USB HID 输入,实时映射到 `Pilot_Cmd_Bus` |

#### 4.7.2 Live passthrough 兼容性约束

未来 Live passthrough 实现必须满足:

1. **输出契约不变**:`Pilot_Cmd_Bus` 字段(per B1 §4.2.1)与 §4.1 字段集合完全一致;消费侧(D4 / D3)无感知差异。
2. **timestamp 来源仍是 harness clock**:外部 RC 设备的硬件时戳**不**直接进入 `Pilot_Cmd_Bus.timestamp`(per A7 §4.6 三模块共享 harness clock)。
3. **deadband / expo 仍由 D4 处理**:Live passthrough 输出 raw stick(与 G4 一致,§4.4),不在 G4 端做 conditioning。
4. **valid 字段反映链路状态**:无 RC 信号时 `valid = 0`(模拟 link loss),与 §4.6.1 / `link_loss` event 行为一致。
5. **Variant 切换由 A5 / G1 拥有**:本文件不规定 Variant 选择机制(VariantSubsystem / configurable subsystem 等),只声明两 variant 在输出契约上必须 swap-in compatible。

#### 4.7.3 Phase 2 不实施

Phase 2 闭环最小切片仅启用 "Scripted CSV" variant;Live passthrough variant 不部署。本节是 forward-cite,不影响 G4 退出条件。

### 4.8 跨引用速查

本节列出 G4 与同 Wave 10 batch 兄弟(G1 / G2 / G3 / E5)以及上游(B1 / B2 / D1 / D4 / A6 / A7 / A2)的接口承诺,作为下游与评审者的速查表:

| 邻接工作项 | G4 提供 / 消费 |
|---|---|
| [G1 MIL 顶层结构设计](G1-mil-toplevel.md)(同 Wave 10) | G4 提供 §4.1 注入字段 + CSV/YAML 文件路径 + ZOH 语义,作为 G1 Pilot_Cmd_Source 子系统的内部行为约定;G1 提供 Pilot_Cmd_Source 子系统块名与位置(known-loose,co-seal 风格,§3 batch 注) |
| [G2 多速率调度设计](G2-rate-scheduling.md)(同 Wave 10) | G4 输出 → RB-06 → FMS,RB-06 块部署由 G2 §4.3 RB-06 行拥有;G4 不重定义 |
| [G3 日志与可观测性设计](G3-logging.md)(同 Wave 10) | G4 注入的 `Pilot_Cmd_Bus` 是 G3 logsout 必采信号(per 计划 §4.G3 "至少包含 INS validity + cmd_mask + mode/state");G3 决定 logsout sample time(per G2 §4.7) |
| [B1 §4.2.1 `Pilot_Cmd_Bus`](../B-contracts/B1-bus-inventory.md) | G4 echo 字段集合;不重定义类型 / 字节宽 / valid 规则 |
| [B2 §4.4.4 `PilotMode`](../B-contracts/B2-enum-inventory.md) | G4 echo 9 成员;CSV `pilot_mode` 列取值集 = enum name 集 |
| [D1 §4.2 命令源](../D-fms/D1-fms-functional.md) | G4 注入 = D1 PILOT 源的输入侧;D1 §4.2.1 行 PILOT(mandatory)对应 G4 必须存在 |
| [D4 §4.3.1 stick conditioning](../D-fms/D4-command-shaper.md) | G4 输出 raw stick,D4 整形(分工见 §4.4)|
| [A6 §4.4 / §4.5.2 init/reset](../A-architecture/A6-init-reset-contract.md) | G4 §4.6 安全态 = HARDCODED zero;G4 不响应 TS-FMS-OUT reset(只响应 TS-MIL-HARNESS) |
| [A7 §4.1–§4.5 时间约定](../A-architecture/A7-time-conventions.md) | G4 timestamp 单位 ms / epoch 零 / 单调非递减 / 不派生 dt(§4.5)|
| [A2 命名](../A-architecture/A2-naming-conventions.md) | 文件名 / 路径 / 列名(`stick_roll` / `_n01` 等)遵守 R-1.x / R-10.x |
| [I5 sim runner](../I-tooling/I5-sim-runner.md)(Wave 12) | G4 声明 CSV / YAML 文件接口;I5 实现 YAML → CSV 预处理 + 装载 |
| [H1 验证场景目录](../H-verification/H1-scenario-catalog.md)(Wave 11) | G4 提供场景文件格式;H1 列出 Phase 2 场景集合并放在 `model/harness/scenarios/<name>/` 下 |
| [H4 故障注入目录](../H-verification/H4-fault-catalog.md)(Wave 11) | G4 §4.3.4 `link_loss` action 是 H4 RC link-loss 故障注入接口的一种;其余 sensor/电池/估计器故障不经过 G4(per §2 不在范围) |

## 5. 已知风险与悬而未决问题

- **G4 与 G1 known-loose co-seal 边界**
  - 影响:orchestrator 把 G1 与 G4 放入同 Wave 10 并行,但关系图 §4.7 标注 `G1 → G4` 强前置。本文件 §3 用 co-seal batch 兄弟规则起草(假定 G1 子系统存在但不依赖具体命名)。若 G1 在评审时决定 Pilot_Cmd_Source 子系统命名 / 信号路由与本文件 §4.1 假设冲突,G4 §4.1 表的"注入到 Pilot_Cmd_Bus 字段"映射不变,但子系统**位置**描述需修订。
  - 处置:G1 reviewer 评审时复核本文件 §4.1 兼容性;若 G1 路由方案需要 G4 改动,本文件追加变更日志。

- **Phase 2 是否需要 DSL?**
  - 影响:§4.3 DSL 是可选高层 generator,Phase 2 闭环最小切片(架构 v1 §16,5 个核心场景)直接手写 CSV 完全够用;DSL 实施会增加 I5 工作量。
  - 处置:Phase 2 决策 = **DSL 接口规范定稿(本文件 §4.3)+ I5 实现推迟**。若 H1 / H4 在 Wave 11 设计场景目录时认为手写 CSV 成本不可接受,启动 I5 DSL 解析器(Wave 12)。开放问题登记。

- **CSV 中 PilotMode 用 name 还是 numeric value 表达**
  - 影响:§4.2.3 选 name(可读 / drift 兼容 / lint 友好);若未来 firmware enum 表大幅扩展,name 集合扩张 reader 必须更新。
  - 处置:本决定 = name;numeric 兜底由 I2 镜像脚本提供(per B5)。reader 在 unknown name 时报错(预防 typo),**不**自动 fallback。

- **CSV 行间 hold-last 与 RC 真实硬件 50–100 Hz 抖动的差距**
  - 影响:CSV 50 Hz 行 + ZOH 看上去 deterministic,但真实 RC 硬件每帧 PWM 解码可能引入 ±1–2 ms 抖动,可能掩盖部分 stick 时序边缘 bug。
  - 处置:推迟到 Phase 5+ Live passthrough(§4.7);Phase 2 故意不模拟该抖动,以最大化 MIL 可重放性(H3 byte-equality 回归)。如 H1 / H4 需要"抖动注入"场景,在 H4 故障目录中显式添加 `stick_jitter` action(本文件 §4.3.4 不含)。

- **场景结尾 hold-last 与 H1 验证窗口**
  - 影响:§4.6.3 选择 hold-last 而非自动归零;H1 / H3 在评估"场景之后"信号时需理解 pilot 输入是 frozen 而非 zero。
  - 处置:本文件已要求场景作者在 CSV 末尾写防御性 disarm 行;G3 logsout 时戳与 G2 调度图(20 ms 主窗口)对齐,使"场景结束 vs 仿真结束"在 logsout 中可区分。本风险 = 文档化注意事项,无需 G4 行为修改。

- **G4 不响应 FMS 触发的 global reset 的副作用**
  - 影响:§4.6.2 决策 G4 不响应 TS-FMS-OUT reset;若 FMS 在场景中段触发 failsafe global reset(TS-FAILSAFE),pilot CSV 继续往前播放,可能导致 pilot 输入在 reset 之后立即又触发 mode 切换,看似"reset 没生效"。
  - 处置:这是 A6 §4.5.2 关键约束 1 的合理推论(FMS 不能 clean 外部输入源);场景作者在设计 failsafe 触发场景时应在 CSV 中显式写入"reset 之后期望的 pilot 输入"(例如全零 + PMODE_NONE 持续若干秒),H4 故障注入目录在场景 spec 中显式标注。

- **本文件未涵盖 contract_impact 风险**
  - 影响:本文件**不**触及 firmware 可见 bus / enum / parameter / symbol(纯模型仓 harness 侧 + 已 reviewed B1 / B2 echo)。`Pilot_Cmd_Bus` schema 与 `PilotMode` 数值由 B1 / B2 拥有;G4 仅消费。
  - 处置:contract_impact = no;无需 INDEX 决策日志登记。若未来 G4 行为修改影响 `Pilot_Cmd_Bus` 字段集(例如新增字段),contract_impact 升级为 yes,通过 B1 修订与 INDEX 决策日志登记。

## 6. 退出条件复核

G4 在 [`00-design-plan.md`](../00-design-plan.md) §4.G 中的退出条件原文:

> 脚本驱动的 Pilot_Cmd 时间序列格式、场景描述 DSL(若需要)

逐条复核:

| # | 退出条件分解 | 本文档依据 | 状态 |
|---|---|---|---|
| 1 | "脚本驱动的 Pilot_Cmd 时间序列格式" | §4.2 主用 CSV 格式定稿:§4.2.1 选型(CSV + 可选 MAT.gz)+ §4.2.2 列定义(15 列含 9 必填 + 8 aux + valid)+ §4.2.3 PilotMode name → enum 转换 + §4.2.4 采样率 50 Hz / hold-last + §4.2.5 文件路径 + §4.2.6 完整示例;§4.5 timestamp 策略(单位 / epoch / 单调 / 不派生 dt)| 满足 |
| 2 | "场景描述 DSL(若需要)" | §4.3 `scenario.yaml` 设计:§4.3.1 引入动机("可选,Phase 2 不强制"对应"若需要")+ §4.3.2 文件位置 + §4.3.3 顶层 schema + §4.3.4 12 个动作枚举 + §4.3.5 完整等价示例 + §4.3.6 DSL ↔ CSV 等价性;Phase 2 决策 = 接口规范定稿 + I5 实现推迟(§5 第二行 risk)| 满足 |

附加(隐含,由 G4 上下文决定):

| # | 隐含条件 | 本文档依据 | 状态 |
|---|---|---|---|
| 3 | 注入字段集合与 B1 §4.2.1 一致(无字段重定义)| §4.1 注入点表 echo B1 §4.2.1 全 10 字段;§2 显式排除 schema 重定义 | 满足 |
| 4 | PilotMode 9 成员与 B2 §4.4.4 一致(无 enum 重定义)| §4.2.2 / §4.2.3 echo B2 9 成员 name;§2 显式排除 enum 重定义 | 满足 |
| 5 | G4 与 D4 stick 整形分工清晰(无越权)| §4.4 整张表把 raw / conditioned / setpoint 三态分配给 G4 / D4 / D4;§2 显式排除 stick 整形 | 满足 |
| 6 | timestamp 与 A7 一致 | §4.5 完整 echo A7 §4.1–§4.5 决策 | 满足 |
| 7 | init / reset 行为与 A6 一致 | §4.6 echo A6 §4.4 类别表 + §4.5.2 reset 协调 | 满足 |
| 8 | Phase 5+ Live passthrough 接口预声明(arch v1 §18 vehicle expansion)| §4.7 三小节(Phase 2 不实施 + 兼容性约束)| 满足 |

退出条件全部满足;状态可由 reviewer 升级到 reviewed。

## 7. 下游影响

按 [`01-design-relationships.md`](../01-design-relationships.md) §4.7:

```text
G1 → G4
```

G4 的下游边在关系图中目前无 outgoing 显式标注(G4 是 leaf 类设计);通过同 Wave / 后续 Wave 间接影响:

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [G1 MIL 顶层结构设计](G1-mil-toplevel.md) *(同 Wave 10 sibling)* | (sibling co-seal) | §4.1 Pilot_Cmd_Source 子系统内部行为(注入字段 + CSV/YAML 文件接口 + ZOH);G1 据此放置 Pilot_Cmd_Source 块 |
| [G3 日志与可观测性设计](G3-logging.md) *(同 Wave 10 sibling)* | (sibling) | `Pilot_Cmd_Bus` 是 G3 logsout 必采信号(per 计划 §4.G3 + 架构 v1 §13);场景文件作为 logsout 命名 metadata 的来源(`<scenario_name>` 出现在 logsout manifest) |
| [I5 sim runner 脚本设计](../I-tooling/I5-sim-runner.md) *(Wave 12)* | ⇢ | §4.2 CSV 格式 + §4.3 YAML schema + §4.3.6 等价性约束 = I5 必须支持的输入格式;I5 实现 YAML → CSV 预处理 + 装载 + 单调性校验 |
| [H1 验证场景目录设计](../H-verification/H1-scenario-catalog.md) *(Wave 11)* | ⇢ | 场景文件位置 / 命名 / 内容格式;H1 在 `model/harness/scenarios/` 下落地具体场景 |
| [H4 故障注入目录设计](../H-verification/H4-fault-catalog.md) *(Wave 11)* | ⇢ | §4.3.4 `link_loss` action 是 RC link-loss 故障类的注入接口(由 G4 拥有);其余故障类不经 G4 |

间接影响:

- **D4 Command Shaper**:G4 §4.4 raw vs conditioned 分工对 D4 §4.3.1 假设的"输入是 [-1, +1] / [0, 1] raw 值"提供 upstream 保证;D4 不需要做 magnitude clipping(已经在 [-1, +1] 内)。
- **D1 §4.2 Command Sources**:G4 是 PILOT 源的实际产出方;D1 §4.2.1 PILOT mandatory 行依赖 G4 在 MIL 路径下提供输入。

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-10 | G4 author | 初稿 |

## Self-check

- [x] frontmatter 完整,字段值合法
- [x] 上游文档全部存在且 status ≥ reviewed(架构 v1: 2026-05-05;A2: reviewed 2026-05-05;A6: reviewed 2026-05-05;A7: reviewed 2026-05-05;B1: reviewed 2026-05-07;B2: reviewed 2026-05-07;D1: reviewed 2026-05-08;D4: reviewed 2026-05-08);G1 同 Wave 10 batch 兄弟,known-loose co-seal 引用,§3 已注明
- [x] 退出条件逐条复核完成,每条均给出依据(§6 共 8 行,2 条退出条件 + 6 条隐含)
- [x] 引用路径全部可点击访问(架构 v1 / A2 / A6 / A7 / B1 / B2 / D1 / D4 / G1 / G2 / G3 / I5 / H1 / H4 / 01-design-relationships)
- [x] 不存在 RULES §5 禁则中的内容(无 .slx 截图;无可执行 .m;CSV / YAML 示例为格式样本而非实现代码;无 firmware 实现细节重述;无 PR/branch 名;无 bus 字段重复定义 — §4.1 全部 echo B1;无 enum 数值重复定义 — §4.2.3 echo B2;依赖列表完整)
- [x] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记 — 本文件 contract_impact=no,N/A
- [x] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径 — 本文件未镜像 firmware 契约(经由 B1 / B2 间接锚定),N/A
- [x] 下游影响已沿关系图识别完毕(§7 含 G1 / G3 / I5 / H1 / H4 + 间接 D4 / D1)
- [x] 文档不超出本工作项范围(`Pilot_Cmd_Bus` 字段重定义留 B1;`PilotMode` 数值留 B2;stick 整形留 D4;Pilot_Cmd_Source 子系统位置留 G1;RB-06 块部署留 G2;logsout 留 G3;sim runner 留 I5;timestamp 单位 / epoch / 回卷留 A7;init/reset 类别留 A6;命名规则留 A2)
