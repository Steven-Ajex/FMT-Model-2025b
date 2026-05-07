---
work_item: B2
title: Enum 清单与数值锁定
upstream: ["架构 v1", "A1", "A2", "A3", "A6", "A7"]
contract_impact: yes
status: draft
authored_at: 2026-05-07
last_reviewed_at:
reviewer_verdict: none
---

# B2 Enum 清单与数值锁定

## 1. 目的

为 FMT-Model-2025b 全部模型仓侧契约 enum(mode / state / status / error / fault / sensor-health / 命令 / mask 位号 / 辅助状态)给出**唯一的数值锁定清单**。本文件是 [B1 Bus 清单与 schema 设计](B1-bus-inventory.md) 与 [B3 Parameter schema 设计](B3-parameter-schema.md) 的 enum-typed 字段所引用的**单一数值真源**(canonical numeric source),并把架构 v1 §17 risk 6 / A1 §5.1.9 标识为 critical 的"silent enum drift"风险落到具体的检测要求上(handoff 给 [B4 契约 diff 策略](B4-contract-diff.md) 与 [I3 契约 diff 脚本](../I-tooling/I3-contract-diff.md))。

## 2. 范围

**在范围:**

- 全部 firmware-visible 契约 enum 的**类型清单**:类型名、底层整型宽度、源 firmware 头文件(相对路径)、所有权侧(firmware-canonical vs model-mirrored)、drift 风险等级。
- 每个契约 enum 的**值表**(name | numeric value | meaning | introduced-in | mirrored-by-model)。其中数值精确锁定的字段标 `(B4 verify)` 表示在 [B4](B4-contract-diff.md) / [I3](../I-tooling/I3-contract-diff.md) 首轮契约 diff 跑通前为**设计意图(design intent)**,实际整数以与 firmware tip 的 diff 结果为准。
- 模型仓侧的 **drift 管理策略**:additive-only、end-of-list、与 firmware lockstep 的规则。
- 给 [B4](B4-contract-diff.md) / [I3](../I-tooling/I3-contract-diff.md) 的 **diff 覆盖要求**(per-name 数值精确相等 / 缺名 = fail / firmware 新增值 = warn)。
- enum 命名规则的引用(完全沿 [A2 §4.6](../A-architecture/A2-naming-conventions.md))。
- `cmd_mask` 位号(bit position)层面的命名 + 位序枚举 — 仅作为 enum 清单条目存在,**不**定义位语义(per [A3 §4.7](../A-architecture/A3-module-boundaries.md))。

**不在范围(由其他工作项处理):**

- enum **字段本身在哪个 bus 出现** — 由 [B1 Bus 清单与 schema 设计](B1-bus-inventory.md) 处理(本文件被 B1 引用)。
- enum **字段在哪个 PARAM struct 出现** — 由 [B3 Parameter schema 设计](B3-parameter-schema.md) 处理(本文件被 B3 引用)。
- `cmd_mask` 各 bit 的**语义**(哪一位代表哪个环路裁剪规则)— 由 [D6 FMS↔Controller 接口约定](../D-fms/D6-fms-controller-interface.md) 与 [E1 Controller 功能设计](../E-controller/E1-controller-functional.md) co-seal batch(Wave 8)处理;本文件按 [A3 §4.7](../A-architecture/A3-module-boundaries.md) 仅列**位号工作名**与底层整型。
- `INS_Out_Bus` 的字段镜像产物本身 — 由 [B5 INS_Out_Bus 镜像同步流程](B5-ins-bus-mirror.md) 在镜像产物中锁定(本文件复述其引用的 enum 类型,标 model-mirrored)。
- enum 值的 ert.tlc 代码生成形态(`enum class` vs `int8_t` typedef)— 由 [I4 Codegen 配置脚本设计](../I-tooling/I4-codegen-config.md) 处理。
- enum 的 mode 集合**功能层语义**(哪些 mode 在哪个 vehicle 启用)— 由 [D1 FMS 功能设计](../D-fms/D1-fms-functional.md) 决定;本文件只锁所有 firmware-visible enum 成员的存在与数值。
- 各模块 fault / health 的**触发条件与阈值** — 由 D / E / C 区各功能设计处理;本文件只锁名称与数值。

## 3. 依赖

| 上游 | 引用位置 | 用途 |
|---|---|---|
| 架构 v1 §5.1.9 | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | "Current enum numeric values are already embedded ... must not drift silently" 是本文件的**契约根**;A1 §5.1.9 把它指派为 B2 + B4 + I3 critical 闭合 |
| 架构 v1 §10 (§10.x mode/state/error 语义) | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | mode/state enum 集中分布说明;B2 严格按"additive-only / end-of-list"规则锁数值 |
| 架构 v1 §14.2 Enum management | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | "Freeze numeric values for any enum already embedded in firmware contract" + "New enum values require explicit compatibility review" + mode-related enum 归属 FMS shared 的总规则 |
| 架构 v1 §17 Risk 6 (silent enum drift) | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | drift 风险与 mitigation 路线 |
| [A1 架构 v1 评审与缺口闭合](../A-architecture/A1-v1-review.md) (status: reviewed, 2026-05-05) | A1 §4.2 §5.1.9 / §14.2;A1 §4.5.3 critical 集中位置 | 本工作项必须闭合的具体缺口 — "未给检测机制" + "FMS shared 与 B 区中央 enum 库归属未明" |
| [A2 命名与目录约定设计](../A-architecture/A2-naming-conventions.md) (status: reviewed, 2026-05-05) | A2 §4.6 R-6.1 / R-6.2 / R-6.3 / R-6.4;A2 §4.8 firmware 已固化对齐表 | enum 类型名 PascalCase + 成员名 UPPER_SNAKE_CASE + 类型短前缀 + 数值锁定 + 归属 `model/shared/enum/` |
| [A3 模块边界信号清单细化](../A-architecture/A3-module-boundaries.md) (status: reviewed, 2026-05-07) | A3 §4.4.4 / §4.5.3 INS 消费侧;§4.4.6.2 控制流 / 状态组;§4.4.6.3 错误组;§4.7 cmd_mask 守卫指针 | 16 个契约 bus 中所有 enum-typed 字段名的来源(`ctrl_mode`、`mode`、`status`、`state`、`ext_state`、`error_code`、`failsafe_state`、`mode_switch`、`cmd_type`、`fix_type`、`INS_Status`、`INS_Flag`)+ `cmd_mask` 位号工作名清单 |
| [A6 Init/Reset 状态契约](../A-architecture/A6-init-reset-contract.md) (status: reviewed, 2026-05-05) | A6 §4.4 类别表;A6 §4.5.3 INS readiness gate | reset-time enum 字段稳态值类别(本文件给"哪个枚举成员代表 reset-time 安全态");A6 列每字段 reset 行为,本文件为 `STATUS_DISARM` / `STATE_DISARM` / `CMODE_*` / `ERR_NONE` / `FIX_NONE` 等"安全 enum 成员"提供数值绑定锚点 |
| [A7 时间与时间戳约定](../A-architecture/A7-time-conventions.md) (status: reviewed) | A7 §4.6 timestamp 字段约束 | 与本文件无直接 enum 交集;仅作为 contract bus 中 timestamp 字段的非 enum 类型佐证(明确 timestamp 不进入本清单) |
| [B1 Bus 清单与 schema 设计](B1-bus-inventory.md) (sibling, **draft in flight, co-seal batch (B1, B2, B3)**) | B1 字段表中所有 enum-typed 字段 | B1 引用本文件的类型名与底层整型宽度;字段顺序由 B1 定 |
| [B3 Parameter schema 设计](B3-parameter-schema.md) (sibling, **draft in flight, co-seal batch (B1, B2, B3)**) | B3 中 enum-typed 参数字段(若有) | B3 引用本文件的类型名;参数字段类型选 enum 时直接使用 |
| `FMT-Firmware/src/model/fms/<vehicle>/lib/FMS_types.h` | `FMT-Firmware @ <pending hash>` | mode / status / state / ext_state / error_code / failsafe_state / cmd_type / mode_switch / ctrl_mode / cmd_mask 位号 等 FMS 侧契约 enum 的源 |
| `FMT-Firmware/src/model/control/<vehicle>/lib/Controller_types.h` | `FMT-Firmware @ <pending hash>` | Controller 侧若有独立 enum(目前主要是 enum 引用 FMS 侧 + 模型内 fault enum)|
| `FMT-Firmware/src/model/plant/<vehicle>/lib/Plant_types.h` | `FMT-Firmware @ <pending hash>` | Plant 侧若有独立 enum(目前主要是模拟传感器健康 enum) |
| `FMT-Firmware/src/model/ins/lib/INS_types.h` | `FMT-Firmware @ <pending hash>` | `INS_Status` / `INS_Flag` 的源(firmware-canonical;本文件 model-mirrored)|

**Co-seal batch 声明**:本工作项属 [01-design-relationships.md §5](../01-design-relationships.md) 设计协同对 **(B1, B2, B3)** — bus / enum / parameter 三件套。Wave 4 同 batch 起草,允许互相引用 draft。本文件依赖的 B1 / B3 在写作时仍是 draft,符合 RULES §6 self-check 第 2 条"或为本工作项所在 co-seal batch 的同批兄弟"。

## 4. 设计内容

### 4.1 Enum 命名约定(echo A2 §4.6)

本文件**完全引用** [A2 §4.6](../A-architecture/A2-naming-conventions.md) 的 R-6.1 / R-6.2 / R-6.3 / R-6.4 / E-6.1 / X-6.1 命名规则。**不重述**。本节只把每条规则映射到本文件值表的列含义:

| A2 规则 | 在本文件中的体现 |
|---|---|
| R-6.1 类型名 PascalCase | §4.4 起每个 per-enum subsection 标题中的 enum 类型名(如 `VehicleStatus`、`PilotMode`、`CtrlMode`、`ErrorCode`、`INS_Status` 等) |
| R-6.2 成员 UPPER_SNAKE_CASE + 类型短前缀 | 每个值表中 "name" 列。前缀采用 A2 §4.6 R-6.2 列出的标准短形:`STATUS_*` / `STATE_*` / `EXT_STATE_*` / `PMODE_*` / `CMODE_*` / `ERR_*` / `FAIL_*` / `MODE_*`(FlightMode)、`FIX_*`、`CMDTYPE_*`、`MISSION_*`、`MASK_BIT_*`、`INS_STATUS_*`、`INS_FLAG_*` |
| R-6.3 数值锁定 | §4.2 drift 管理政策 + 各值表的 numeric value 列;**这是本文件存在的核心** |
| R-6.4 命名归属 (`model/shared/enum/`) | §4.4 起所有契约 enum 一律归 `model/shared/enum/<EnumName>.m`(或等价 SLDD 条目);不分散到 vehicle leaf |
| E-6.1 firmware 已固化成员名按现状保留 | 各值表标 `(B4 verify)` 列;若 firmware 实际成员名不符合 A2 风格,以 firmware 为准并在本文件值表 "name" 列改为 firmware 实名(B4 首跑后定稿) |

**本文件不**新引入命名风格;任何与 A2 §4.6 风格的差异都视为 firmware-driven 例外,在表内单独标注。

### 4.2 Drift 管理政策(附加性 / 末尾追加 / 与 firmware lockstep)

本节为 A1 §5.1.9 critical 缺口"未给检测机制"提供策略层闭合;实操层由 [B4](B4-contract-diff.md) 策略 + [I3](../I-tooling/I3-contract-diff.md) 脚本兑现。

**P-1 不可变值约束(immutable values)**:任一已在本清单中数值锁定的 enum 成员,在模型仓侧**永不**重新分配数值;包括但不限于:

- 不交换两个成员的数值
- 不在已存在成员的"中间"插入新成员(任何插入都必须 append-only)
- 不重命名已固化的成员(firmware 重命名时由 B4 警示并单独走变更评审)
- 不删除已固化的成员(firmware 弃用走 deprecated 标记 + 末尾保留;详见 P-4)

**P-2 末尾追加规则(end-of-list, additive-only)**:新增 enum 成员只能追加在该 enum 当前最大数值之后,数值 = 当前最大值 + 1。**禁止**填补由 deprecate 留下的"洞"。

**P-3 lockstep 规则(与 firmware 同步)**:模型仓侧 enum 任何变更(无论新增 / 重命名 / deprecate)**必须**满足以下其一:

1. **firmware 已先行变更**,模型仓在同一次镜像周期(per [B5 INS_Out_Bus 镜像同步流程](B5-ins-bus-mirror.md) 类比规则,扩展到 FMS / Controller / Plant 侧 enum)同步 + B4 diff pass。
2. **模型仓发起变更**,在合入前由人发起 firmware 提案,等待 firmware tip 落地,再走第 1 条流程。
3. 严禁"模型仓单方面增 / 改 enum 数值"——这是 silent drift 的直接来源(架构 v1 §17 Risk 6)。

**P-4 deprecate 流程**:firmware 弃用某 enum 成员时,模型仓**保留**该成员名与原数值(不删),在该成员行的 "introduced-in" / Notes 列加 `DEPRECATED at <firmware version>`,新代码不得使用,但二进制层面保持兼容。

**P-5 enum 底层整型宽度变更**(例如 uint8 → uint16)被视为**契约破坏性**变更,需走 firmware 主导的契约破坏流程,B4 必须 fail,并在架构 v1 §17 Risk 6 风险记录中追加。

**P-6 默认 firmware-canonical**:所有列于 §4.4 的契约 enum 默认所有权 = firmware-canonical(模型仓 mirrored)。**例外**:若某 enum 由模型仓侧 FMS shared 模块**首发**(架构 v1 §14.2 "Mode-related enums should be owned by FMS shared definitions"),则所有权 = model-canonical;具体 enum 在 §4.4 各 subsection header 中标注。

### 4.3 Diff 覆盖要求(给 B4 / I3 的验收标准)

本节是 A1 §5.1.9 把闭合责任分给 B2 + B4 + I3 三方时,B2 应**输出**给 B4 / I3 的输入规范。详细 diff 工具输入输出格式由 [B4](B4-contract-diff.md) / [I3](../I-tooling/I3-contract-diff.md) 自行定;本节给"必须覆盖什么"。

**D-1 per-enum-type 类型存在性检查**:本文件 §4.4 列出的每个 enum 类型名必须在 firmware `*_types.h` 中存在(typedef 或 enum class 形式);缺失 = **fail**。

**D-2 per-name 数值精确相等**:本文件每个 enum 值表中标 `mirrored-by-model = yes` 的成员,其 (name, numeric_value) 二元组必须与 firmware 完全相等;不等 = **fail**。

**D-3 缺名检测**:本文件值表中存在但 firmware 不存在的成员名 = **fail**(意味着模型仓凭空增了 enum 值,违反 P-3)。

**D-4 firmware-only 成员检测**:firmware 中存在但本文件值表未列的成员名 = **warn**(意味着 firmware 新增了 enum 值,本文件需走 P-2 同步;在同步前,B4 仍 warn 不 fail,以避免 firmware 单边推进时模型仓 CI 长期 red)。

**D-5 底层宽度检查**:每个 enum 底层 underlying type(uint8 / uint16 / uint32)必须与 firmware `*_types.h` 中该 enum 的 sizeof(在生成代码中等价的整数宽度)相等;不等 = **fail**(per P-5)。

**D-6 引入版本一致性(可选,non-blocking)**:本文件 "introduced-in" 列若已标具体 firmware 版本号,B4 在条件允许时(firmware 提供历史信息)可比对;不一致 = **warn**(non-blocking,以免历史信息缺失阻塞 CI)。

**D-7 输出格式**:B4 / I3 报告必须能定位到 (enum_type, member_name, model_value, firmware_value, verdict) 五元组,以便 reviewer 一眼定位 drift 位置。

**D-8 触发时机**:B4 §3 给出运行时机(导出前 + CI),本文件不重述;只声明本文件被 [I2 Bus/enum 镜像脚本](../I-tooling/I2-bus-enum-mirror.md) 作为镜像目标参考之一。

### 4.4 Per-enum 目录(契约 enum 总清单)

本节为本文件主体。每个 subsection 一个 enum 类型。每个 subsection 包含:

- **Header 表**:类型名、underlying type、源 firmware 头文件、所有权侧、drift 风险、引用本 enum 的契约 bus / PARAM 字段(指针给 [B1](B1-bus-inventory.md) / [B3](B3-parameter-schema.md))。
- **值表**:name | numeric value | meaning | introduced-in | mirrored-by-model。
- **Notes**:gap fields、deprecate 状态、与其他 enum 的关系。

> **数值占位说明**:在没有 mount FMT-Firmware 文件系统的当前条件下,本文件的具体整数值是 **设计意图(design intent)**,以 PX4-style + 现有 FMT-Firmware mc_fms 行为模式为合理默认。所有标 `(B4 verify)` 的整数,**首轮 [B4](B4-contract-diff.md) / [I3](../I-tooling/I3-contract-diff.md) 跑通后以 firmware tip 为准定稿**;本文件届时按 §10 变更日志规则更新。

#### 4.4.1 `VehicleStatus` — 飞行器整体状态(top-level)

| 属性 | 值 |
|---|---|
| 类型名 | `VehicleStatus` |
| Underlying type | `uint8` `(B4 verify)` |
| 源 firmware 头 | `FMT-Firmware/src/model/fms/<vehicle>/lib/FMS_types.h` |
| 所有权侧 | firmware-canonical(P-6 默认),mode 系列由 FMS shared 拥有(架构 v1 §14.2;模型仓侧 mirrored) |
| Drift 风险 | **High**(架构 v1 §17 Risk 6;A3 §4.4.6.2 `status` 字段直接进入 `FMS_Out_Bus`) |
| 被引用位置 | `FMS_Out_Bus.status`(B1);`FMS_PARAM` 内若有 `default_status`(B3) |

| name | numeric value | meaning | introduced-in | mirrored-by-model |
|---|---|---|---|---|
| `STATUS_DISARM` | `0` `(B4 verify)` | 整机未解锁(safe / power-on default);A6 §4.4 reset-time disarmed-equiv 默认锚定到此 | current firmware tip | yes |
| `STATUS_STANDBY` | `1` `(B4 verify)` | 已通电、自检通过、等待解锁 | current firmware tip | yes |
| `STATUS_ARM` | `2` `(B4 verify)` | 已解锁,可执行飞行控制 | current firmware tip | yes |

Notes:

- A6 §4.4 reset-time "HARDCODED disarmed-equiv" 在本 enum 中实现 = `STATUS_DISARM`(数值 0);若 firmware 实际选其它成员作为安全态,B4 验证 + 本文件 §10 变更日志同步。
- 与 `VehicleState`(§4.4.2)是**正交**两个 enum:status 描述 arming 状态,state 描述飞行阶段。Mode Manager(D3)在状态机中同时输出两者。
- 若 firmware 增加了 `STATUS_*` 新成员(例如 `STATUS_CALIBRATING`),按 P-2 末尾追加 = `3` 起。

#### 4.4.2 `VehicleState` — 飞行阶段子状态

| 属性 | 值 |
|---|---|
| 类型名 | `VehicleState` |
| Underlying type | `uint8` `(B4 verify)` |
| 源 firmware 头 | `FMT-Firmware/src/model/fms/<vehicle>/lib/FMS_types.h` |
| 所有权侧 | firmware-canonical(mode 系列 FMS shared) |
| Drift 风险 | **High**(契约 bus `FMS_Out_Bus.state`)|
| 被引用位置 | `FMS_Out_Bus.state`(B1)|

| name | numeric value | meaning | introduced-in | mirrored-by-model |
|---|---|---|---|---|
| `STATE_DISARM` | `0` `(B4 verify)` | 与 `STATUS_DISARM` 配对的 state;reset-time 默认 | current firmware tip | yes |
| `STATE_STANDBY` | `1` `(B4 verify)` | 待机 | current firmware tip | yes |
| `STATE_LOCKDOWN` | `2` `(B4 verify)` | 锁定(error / kill-switch 触发后的安全态) | current firmware tip | yes |
| `STATE_INAIR` | `3` `(B4 verify)` | 在空中(已脱离地面)| current firmware tip | yes |
| `STATE_GROUND` | `4` `(B4 verify)` | 在地面(已落地或未起飞但已 arm)| current firmware tip | yes |

Notes:

- D3 Mode Manager 状态机以 `(VehicleStatus, VehicleState)` 二元组作为 mode/state 输出锚点;本文件锁定数值,D3 锁定状态机转换。
- 若 firmware 把 `STATE_INAIR` / `STATE_GROUND` 进一步细分(如 `STATE_TAKEOFF` / `STATE_LAND`),按 P-2 追加。

#### 4.4.3 `VehicleExtState` — 扩展状态(细化)

| 属性 | 值 |
|---|---|
| 类型名 | `VehicleExtState` |
| Underlying type | `uint8` `(B4 verify)` |
| 源 firmware 头 | `FMT-Firmware/src/model/fms/<vehicle>/lib/FMS_types.h` |
| 所有权侧 | firmware-canonical |
| Drift 风险 | **Medium**(契约 bus `FMS_Out_Bus.ext_state`,但 ext_state 多用于 logging / GCS 显示,功能裁决较少)|
| 被引用位置 | `FMS_Out_Bus.ext_state`(B1)|

| name | numeric value | meaning | introduced-in | mirrored-by-model |
|---|---|---|---|---|
| `EXT_STATE_LANDED` | `0` `(B4 verify)` | 在地;reset-time 默认 | current firmware tip | yes |
| `EXT_STATE_TAKING_OFF` | `1` `(B4 verify)` | 起飞中 | current firmware tip | yes |
| `EXT_STATE_HOLDING` | `2` `(B4 verify)` | 悬停 / 等待 | current firmware tip | yes |
| `EXT_STATE_MOVING` | `3` `(B4 verify)` | 巡航 / 移动中 | current firmware tip | yes |
| `EXT_STATE_RETURNING` | `4` `(B4 verify)` | RTL 中 | current firmware tip | yes |
| `EXT_STATE_LANDING` | `5` `(B4 verify)` | 降落中 | current firmware tip | yes |

Notes:

- A3 §4.4.6.2 `ext_state` 字段被列为 enum;本文件锁定其类型与值集。
- D3 状态机不需在 `ext_state` 上做 mode 抉择,仅做"信息"输出。

#### 4.4.4 `PilotMode` — 飞手 / 模式开关位置

| 属性 | 值 |
|---|---|
| 类型名 | `PilotMode` |
| Underlying type | `uint8` `(B4 verify)` |
| 源 firmware 头 | `FMT-Firmware/src/model/fms/<vehicle>/lib/FMS_types.h` |
| 所有权侧 | firmware-canonical |
| Drift 风险 | **High**(契约 bus 输入 `Pilot_Cmd_Bus.mode_switch`,任何错位都会让飞手意图被错误解读)|
| 被引用位置 | `Pilot_Cmd_Bus.mode_switch`(B1)|

| name | numeric value | meaning | introduced-in | mirrored-by-model |
|---|---|---|---|---|
| `PMODE_NONE` | `0` `(B4 verify)` | 无模式 / 开关脱出 / 默认 | current firmware tip | yes |
| `PMODE_MANUAL` | `1` `(B4 verify)` | 手动姿态 | current firmware tip | yes |
| `PMODE_STABILIZE` | `2` `(B4 verify)` | 稳定(自调平 attitude)| current firmware tip | yes |
| `PMODE_ALTHOLD` | `3` `(B4 verify)` | 高度保持 | current firmware tip | yes |
| `PMODE_POSHOLD` | `4` `(B4 verify)` | 位置保持 | current firmware tip | yes |
| `PMODE_MISSION` | `5` `(B4 verify)` | 任务模式 | current firmware tip | yes |
| `PMODE_RTL` | `6` `(B4 verify)` | 自动返航 | current firmware tip | yes |
| `PMODE_LAND` | `7` `(B4 verify)` | 自动降落 | current firmware tip | yes |
| `PMODE_OFFBOARD` | `8` `(B4 verify)` | Offboard / 外部脚本接管 | current firmware tip | yes |

Notes:

- D1 FMS 功能设计将基于本 enum 决定具体启用 mode 集(架构 v1 §14.2 + A1 §4.5.3 OQ2);本文件锁定**全集**,D1 决定**子集**。
- `PilotMode` 与 `FlightMode`(§4.4.5)的关系:`PilotMode` 是 RC / GCS 输入侧;`FlightMode` 是 FMS 经过 mode 仲裁后输出到 `FMS_Out_Bus.mode` 侧。两者在多旋翼上多有重叠,但 firmware 仍保留两个 enum,本文件如实保留。

#### 4.4.5 `FlightMode` — FMS 输出 mode

| 属性 | 值 |
|---|---|
| 类型名 | `FlightMode` |
| Underlying type | `uint8` `(B4 verify)` |
| 源 firmware 头 | `FMT-Firmware/src/model/fms/<vehicle>/lib/FMS_types.h` |
| 所有权侧 | firmware-canonical |
| Drift 风险 | **High**(契约 bus `FMS_Out_Bus.mode` — Controller / GCS / log 都消费此字段)|
| 被引用位置 | `FMS_Out_Bus.mode`(B1)|

| name | numeric value | meaning | introduced-in | mirrored-by-model |
|---|---|---|---|---|
| `MODE_NONE` | `0` `(B4 verify)` | 无 mode / 默认 disarmed-equiv | current firmware tip | yes |
| `MODE_MANUAL` | `1` `(B4 verify)` | 手动 | current firmware tip | yes |
| `MODE_STABILIZE` | `2` `(B4 verify)` | 自稳 | current firmware tip | yes |
| `MODE_ALTHOLD` | `3` `(B4 verify)` | 高度保持 | current firmware tip | yes |
| `MODE_POSHOLD` | `4` `(B4 verify)` | 位置保持 | current firmware tip | yes |
| `MODE_MISSION` | `5` `(B4 verify)` | 任务 | current firmware tip | yes |
| `MODE_RTL` | `6` `(B4 verify)` | 返航 | current firmware tip | yes |
| `MODE_LAND` | `7` `(B4 verify)` | 降落 | current firmware tip | yes |
| `MODE_OFFBOARD` | `8` `(B4 verify)` | Offboard | current firmware tip | yes |
| `MODE_ACRO` | `9` `(B4 verify)` | 特技 / 角速率直通 | current firmware tip | yes |

Notes:

- 与 `PilotMode` 在数值上**不保证一致**(firmware 的 `PilotMode` 与 `FlightMode` 是两个独立 enum);D3 状态机做 `PilotMode → FlightMode` 映射。本文件不假设两个 enum 数值相等。
- A6 §4.4 reset-time `mode` 字段安全态 = `MODE_NONE`(数值 0)。

#### 4.4.6 `CtrlMode` — Controller 外部可见控制律模式

| 属性 | 值 |
|---|---|
| 类型名 | `CtrlMode` |
| Underlying type | `uint8` `(B4 verify)` |
| 源 firmware 头 | `FMT-Firmware/src/model/fms/<vehicle>/lib/FMS_types.h`(由 FMS 推出);Controller 消费 |
| 所有权侧 | firmware-canonical(由 FMS 拥有,Controller mirror)|
| Drift 风险 | **High**(契约 bus `FMS_Out_Bus.ctrl_mode` + `Control_Out_Bus.ctrl_mode` echo,直接决定 Controller 控制律分支)|
| 被引用位置 | `FMS_Out_Bus.ctrl_mode`(B1);`Control_Out_Bus.ctrl_mode` echo(B1) |

| name | numeric value | meaning | introduced-in | mirrored-by-model |
|---|---|---|---|---|
| `CMODE_DISARMED` | `0` `(B4 verify)` | Controller idle / 不输出有效控制 | current firmware tip | yes |
| `CMODE_MANUAL_ANGLE` | `1` `(B4 verify)` | 角度环作为入口(手动姿态)| current firmware tip | yes |
| `CMODE_MANUAL_RATE` | `2` `(B4 verify)` | 角速率环作为入口(ACRO)| current firmware tip | yes |
| `CMODE_AUTO_VEL` | `3` `(B4 verify)` | 速度环作为入口(altitude-hold / position-hold)| current firmware tip | yes |
| `CMODE_AUTO_POS` | `4` `(B4 verify)` | 位置环作为入口(POSHOLD / MISSION)| current firmware tip | yes |
| `CMODE_THROTTLE_PASSTHROUGH` | `5` `(B4 verify)` | 油门直通(MANUAL passthrough)| current firmware tip | yes |

Notes:

- A3 §4.4.6.2 `ctrl_mode` enum 直接进入 `FMS_Out_Bus`,Controller 据其选择控制律分支;由 [E1 Controller 功能设计](../E-controller/E1-controller-functional.md)(co-seal Wave 8 with D6)落到具体环路启用裁剪规则。本文件**只**锁数值,不锁裁剪规则。
- A6 §4.4 reset-time `ctrl_mode` 字段安全态 = `CMODE_DISARMED`(数值 0)。

#### 4.4.7 `FailsafeState` — failsafe 子状态

| 属性 | 值 |
|---|---|
| 类型名 | `FailsafeState` |
| Underlying type | `uint8` `(B4 verify)` |
| 源 firmware 头 | `FMT-Firmware/src/model/fms/<vehicle>/lib/FMS_types.h` |
| 所有权侧 | firmware-canonical |
| Drift 风险 | **Medium**(`FMS_Out_Bus.failsafe_state` 在 A3 §4.4.6.3 标 *(optional)*;若 firmware 含此字段,本 enum 即契约 enum)|
| 被引用位置 | `FMS_Out_Bus.failsafe_state`(B1,optional;B1 锁定其存在性)|

| name | numeric value | meaning | introduced-in | mirrored-by-model |
|---|---|---|---|---|
| `FAIL_NONE` | `0` `(B4 verify)` | 无 failsafe;reset-time 默认 | current firmware tip | yes |
| `FAIL_RC_LOSS` | `1` `(B4 verify)` | RC 链路丢失 | current firmware tip | yes |
| `FAIL_GCS_LOSS` | `2` `(B4 verify)` | GCS 链路丢失 | current firmware tip | yes |
| `FAIL_LOW_BATTERY` | `3` `(B4 verify)` | 低电量 | current firmware tip | yes |
| `FAIL_GEOFENCE` | `4` `(B4 verify)` | 越界 | current firmware tip | yes |
| `FAIL_INS_INVALID` | `5` `(B4 verify)` | INS 失效(`INS_Status.ready=0` 或关键 flag.* 长期 0)| current firmware tip | yes |
| `FAIL_OFFBOARD_LOSS` | `6` `(B4 verify)` | Offboard 流丢失 | current firmware tip | yes |

Notes:

- 触发阈值与 transition 由 [D1 FMS 功能设计](../D-fms/D1-fms-functional.md) Safety Monitor 子模块决定(A1 §12.1.3);本文件只锁 enum 数值。
- A6 §4.4 reset-time `failsafe_state` 字段安全态 = `FAIL_NONE`(数值 0)。

#### 4.4.8 `MissionState` — 任务执行子状态

| 属性 | 值 |
|---|---|
| 类型名 | `MissionState` |
| Underlying type | `uint8` `(B4 verify)` |
| 源 firmware 头 | `FMT-Firmware/src/model/fms/<vehicle>/lib/FMS_types.h` |
| 所有权侧 | firmware-canonical |
| Drift 风险 | **Low–Medium**(mission 子状态主要内部 + log;若 firmware 把它纳入 `FMS_Out_Bus`,升为 Medium。本文件按"内部 + 可选契约"处理)|
| 被引用位置 | (若 B1 在 `FMS_Out_Bus` 中含 `mission_state` 字段则适用,否则仅作为 D 区内部 enum 引用 — 由 B1 锁定存在性)|

| name | numeric value | meaning | introduced-in | mirrored-by-model |
|---|---|---|---|---|
| `MISSION_IDLE` | `0` `(B4 verify)` | 无任务;reset-time 默认 | current firmware tip | yes(若契约存在)|
| `MISSION_LOADED` | `1` `(B4 verify)` | 任务已加载 / 待启动 | current firmware tip | yes |
| `MISSION_ACTIVE` | `2` `(B4 verify)` | 任务执行中 | current firmware tip | yes |
| `MISSION_PAUSED` | `3` `(B4 verify)` | 任务已暂停 | current firmware tip | yes |
| `MISSION_COMPLETE` | `4` `(B4 verify)` | 任务正常结束 | current firmware tip | yes |
| `MISSION_ABORTED` | `5` `(B4 verify)` | 任务中止(failsafe 或人工)| current firmware tip | yes |

Notes:

- 范围 / 子集裁剪由 [D1](../D-fms/D1-fms-functional.md) + [D5](../D-fms/D5-multicopter-leaf.md) 处理;架构 v1 §16 把 mission 限制在闭环切片必需子集。本文件锁全集数值,D1 决定使用子集。

#### 4.4.9 `ErrorCode` — 飞行器错误代码

| 属性 | 值 |
|---|---|
| 类型名 | `ErrorCode` |
| Underlying type | `uint8` `(B4 verify)`(若错误码超过 256 项可能升 uint16,首跑 B4 验证)|
| 源 firmware 头 | `FMT-Firmware/src/model/fms/<vehicle>/lib/FMS_types.h` |
| 所有权侧 | firmware-canonical |
| Drift 风险 | **High**(`FMS_Out_Bus.error*` 直接进入契约;架构 v1 §5.1.5 提到 error 字段)|
| 被引用位置 | `FMS_Out_Bus.error_code`(A3 §4.4.6.3;B1 锁定字段名)|

| name | numeric value | meaning | introduced-in | mirrored-by-model |
|---|---|---|---|---|
| `ERR_NONE` | `0` `(B4 verify)` | 无错误;reset-time 默认 | current firmware tip | yes |
| `ERR_INS_INVALID` | `1` `(B4 verify)` | INS 不可用 | current firmware tip | yes |
| `ERR_GPS_LOST` | `2` `(B4 verify)` | GPS 失效 | current firmware tip | yes |
| `ERR_BATTERY_LOW` | `3` `(B4 verify)` | 电量低 | current firmware tip | yes |
| `ERR_BATTERY_CRITICAL` | `4` `(B4 verify)` | 电量临界 | current firmware tip | yes |
| `ERR_RC_LOST` | `5` `(B4 verify)` | RC 丢失 | current firmware tip | yes |
| `ERR_GEOFENCE` | `6` `(B4 verify)` | 越界 | current firmware tip | yes |
| `ERR_MOTOR_FAULT` | `7` `(B4 verify)` | 电机故障(由 Plant / Controller 推出)| current firmware tip | yes |
| `ERR_MODE_INVALID` | `8` `(B4 verify)` | mode 切换前提不满足(D3 Mode Manager 拒绝)| current firmware tip | yes |
| `ERR_OFFBOARD_TIMEOUT` | `9` `(B4 verify)` | Offboard 超时 | current firmware tip | yes |
| `ERR_INTERNAL` | `10` `(B4 verify)` | 内部错误(unspecified)| current firmware tip | yes |

Notes:

- A6 §4.4 reset-time `error_code` 字段安全态 = `ERR_NONE`。
- 架构 v1 §5.1.5 列出的 `error` 字段在本 enum 上落地。
- 触发条件与每个 ERR 的具体动作由 D1 Safety Monitor + D / E 区故障处理 + H4 故障注入目录决定。本文件**只**锁名称 + 数值。

#### 4.4.10 `INS_Status` — INS 总体状态(firmware-canonical;模型仓 mirrored)

| 属性 | 值 |
|---|---|
| 类型名 | `INS_Status` |
| Underlying type | `uint8` `(B4 verify)`(若 firmware 用 bitfield wrapper,可能为 `uint16`;首跑 B4 验证)|
| 源 firmware 头 | `FMT-Firmware/src/model/ins/lib/INS_types.h` |
| 所有权侧 | **firmware-canonical**(INS 由 firmware 拥有,见架构 v1 §5.1.8 / §12.3;模型仓**只**镜像;镜像流程由 [B5](B5-ins-bus-mirror.md) 落实)|
| Drift 风险 | **High**(任何 INS_Status 错位都会导致 FMS / Controller 错误 gate;架构 v1 §17 Risk 2 INS_Out_Bus contract drift)|
| 被引用位置 | `INS_Out_Bus.INS_Status`(A3 §4.4.4 / §4.5.3;B1 / B5 锁定字段名 / 字节布局)|

| name | numeric value | meaning | introduced-in | mirrored-by-model |
|---|---|---|---|---|
| `INS_STATUS_NOT_READY` | `0` `(B4 verify)` | INS 未就绪;A6 §4.5.3 init 后必须 gate `ready` 位 | current firmware tip | yes |
| `INS_STATUS_INITIALIZING` | `1` `(B4 verify)` | INS 自初始化中 | current firmware tip | yes |
| `INS_STATUS_READY` | `2` `(B4 verify)` | INS 就绪;`ready` 位等价语义 | current firmware tip | yes |
| `INS_STATUS_DEGRADED` | `3` `(B4 verify)` | 部分 flag 失效但仍提供降级输出 | current firmware tip | yes |
| `INS_STATUS_FAULT` | `4` `(B4 verify)` | INS 故障 | current firmware tip | yes |

Notes:

- A3 §4.4.4 把 INS_Status 列为"`status` bitfield 或 enum,per firmware (B5 to mirror)";本文件**按 enum 形式**锁定;若 [B5](B5-ins-bus-mirror.md) 镜像 firmware 实际是 bitfield,本文件 §10 变更日志同步切换 — 但 enum value 与 bitfield 取值在数值上**应当**保持一致以便 B4 单一规则覆盖。
- D / E 模块对此 enum 的"ready" 语义判定:`INS_Status >= INS_STATUS_READY` 视为 ready(`flag` bitfield 中的 `position_valid` / `attitude_valid` / `velocity_valid` 位独立检查,见 §4.4.11)。
- A6 §4.5.3 INS readiness gate 默认 `INS_STATUS_NOT_READY` 作 reset-time 安全态。

#### 4.4.11 `INS_Flag` — INS 字段 validity 位枚举(bitfield)

| 属性 | 值 |
|---|---|
| 类型名 | `INS_Flag`(模型仓侧把 firmware 的 flag bitfield **位号**列为 enum;字段类型本身在 B1 是 `uint32` bitfield)|
| Underlying type | `uint32` bitfield(此 enum 列出**位号**:0..N;实际 bus 字段类型 uint32 由 B1 锁)|
| 源 firmware 头 | `FMT-Firmware/src/model/ins/lib/INS_types.h` |
| 所有权侧 | firmware-canonical |
| Drift 风险 | **High**(每个 validity 位的语义错位会让 FMS / Controller 在错误位置 gate;直接进入 `INS_Out_Bus.flag`)|
| 被引用位置 | `INS_Out_Bus.flag`(A3 §4.4.4;B1 锁字段名,B5 锁字节布局)|

值表(每行 = 1 个位号 = 1 个 enum 成员):

| name | numeric value (= bit position) | meaning | introduced-in | mirrored-by-model |
|---|---|---|---|---|
| `INS_FLAG_BIT_POSITION_VALID` | `0` `(B4 verify)` | NED 位置有效;A3 §4.4.4 FMS Mode Mgr / Mission / Pos Loop gate | current firmware tip | yes |
| `INS_FLAG_BIT_VELOCITY_VALID` | `1` `(B4 verify)` | NED 速度有效;Vel Loop / Shaper gate | current firmware tip | yes |
| `INS_FLAG_BIT_ATTITUDE_VALID` | `2` `(B4 verify)` | 姿态有效;Mode Mgr / Att Loop gate | current firmware tip | yes |
| `INS_FLAG_BIT_HEADING_VALID` | `3` `(B4 verify)` | 偏航 / 航向有效 | current firmware tip | yes |
| `INS_FLAG_BIT_MAG_VALID` | `4` `(B4 verify)` | 磁力计输入有效(进入估计器)| current firmware tip | yes |
| `INS_FLAG_BIT_GPS_VALID` | `5` `(B4 verify)` | GPS 输入有效;Safety Monitor gate | current firmware tip | yes |
| `INS_FLAG_BIT_BARO_VALID` | `6` `(B4 verify)` | 气压计输入有效 | current firmware tip | yes |
| `INS_FLAG_BIT_AIRSPEED_VALID` | `7` `(B4 verify)` | 空速输入有效(固定翼 / VTOL)| current firmware tip | yes |
| `INS_FLAG_BIT_VEL_BODY_VALID` | `8` `(B4 verify)` | body-frame 速度有效(若 firmware 有此项)| current firmware tip | yes |

Notes:

- 本 enum 的"numeric value" 列就是**位号**(0..31),不是位掩码值(`1<<bit`)。生成代码中如何使用该位号是 D / E / B1 决定:可直接位移,或通过宏 `INS_FLAG_MASK(bit) = (1u << (bit))` 包装。
- 模型仓内部 `Validity_AND` 原语(A8 §4.2.5)消费这些位号;本文件只锁位号编排。
- 若 firmware `INS_Flag` 是 enum class 而非 bitfield 集合,**本文件仍**用相同位号语义(每位一个 enum 成员);B5 镜像产物如实保留 firmware 形式。
- 总位数 ≤ 32(底层 uint32 限制);超过即触发 P-5(底层宽度变更),需走破坏性流程。

#### 4.4.12 `GPS_FixType` — GPS 定位类型(uBlox)

| 属性 | 值 |
|---|---|
| 类型名 | `GPS_FixType` |
| Underlying type | `uint8` `(B4 verify)` |
| 源 firmware 头 | `FMT-Firmware/src/model/plant/<vehicle>/lib/Plant_types.h`(simulated GPS;real GPS 在 firmware INS 侧)|
| 所有权侧 | firmware-canonical(uBlox 协议数值固化)|
| Drift 风险 | **Medium**(uBlox 协议数值是工业标准;主要风险在 firmware 侧选择保留哪些等级)|
| 被引用位置 | `GPS_uBlox_Bus.fix_type`(A3 §4.3.7.4;B1 锁字段名)|

| name | numeric value | meaning | introduced-in | mirrored-by-model |
|---|---|---|---|---|
| `FIX_NONE` | `0` `(B4 verify)` | 无定位;reset-time 默认 | current firmware tip | yes |
| `FIX_DEAD_RECKONING` | `1` `(B4 verify)` | 仅航位推算 | current firmware tip | yes |
| `FIX_2D` | `2` `(B4 verify)` | 2D 定位 | current firmware tip | yes |
| `FIX_3D` | `3` `(B4 verify)` | 3D 定位 | current firmware tip | yes |
| `FIX_GNSS_DEAD_RECKONING` | `4` `(B4 verify)` | GNSS + 推算融合 | current firmware tip | yes |
| `FIX_TIME_ONLY` | `5` `(B4 verify)` | 仅时间同步 | current firmware tip | yes |

Notes:

- 数值取自 uBlox 协议惯例;若 firmware 侧已有重映射,B4 首跑后定稿。
- A6 §4.4 reset-time `fix_type` 字段安全态 = `FIX_NONE`。

#### 4.4.13 `GCS_CmdType` — GCS 命令类别

| 属性 | 值 |
|---|---|
| 类型名 | `GCS_CmdType` |
| Underlying type | `uint8` `(B4 verify)` |
| 源 firmware 头 | `FMT-Firmware/src/model/fms/<vehicle>/lib/FMS_types.h`(GCS_Cmd_Bus)|
| 所有权侧 | firmware-canonical |
| Drift 风险 | **Medium**(GCS 输入侧;错位影响命令解读但不直接进入闭环关键 gate)|
| 被引用位置 | `GCS_Cmd_Bus.cmd_type`(A3 §4.4.3.1;B1 锁字段名)|

| name | numeric value | meaning | introduced-in | mirrored-by-model |
|---|---|---|---|---|
| `CMDTYPE_IDLE` | `0` `(B4 verify)` | 无命令;reset-time 默认 | current firmware tip | yes |
| `CMDTYPE_ARM` | `1` `(B4 verify)` | 解锁 | current firmware tip | yes |
| `CMDTYPE_DISARM` | `2` `(B4 verify)` | 锁定 | current firmware tip | yes |
| `CMDTYPE_TAKEOFF` | `3` `(B4 verify)` | 起飞 | current firmware tip | yes |
| `CMDTYPE_LAND` | `4` `(B4 verify)` | 降落 | current firmware tip | yes |
| `CMDTYPE_RTL` | `5` `(B4 verify)` | 返航 | current firmware tip | yes |
| `CMDTYPE_MODE_CHANGE` | `6` `(B4 verify)` | 模式切换(参数携带目标 mode)| current firmware tip | yes |
| `CMDTYPE_SETPOINT_UPDATE` | `7` `(B4 verify)` | setpoint 更新(参数携带 lat/lon/alt 或 NED 偏置)| current firmware tip | yes |
| `CMDTYPE_SET_HOME` | `8` `(B4 verify)` | 设置 home | current firmware tip | yes |
| `CMDTYPE_PAUSE_MISSION` | `9` `(B4 verify)` | 任务暂停 | current firmware tip | yes |
| `CMDTYPE_RESUME_MISSION` | `10` `(B4 verify)` | 任务继续 | current firmware tip | yes |
| `CMDTYPE_OFFBOARD` | `11` `(B4 verify)` | Offboard 切入 | current firmware tip | yes |

Notes:

- A3 §4.4.3.1 `cmd_param[]` 长度由 B1 锁;参数语义随 `cmd_type` 变,语义层映射由 D1 / D2 处理。本文件只锁 enum 数值。

#### 4.4.14 `cmd_mask` 位号清单(bit-position 工作名 + 数值;**位语义留 D6/E1**)

| 属性 | 值 |
|---|---|
| 类型名 | `CmdMaskBit`(模型仓侧 bit-position 枚举;**字段类型本身**在 `FMS_Out_Bus.cmd_mask` 是 `uint32` bitfield,由 B1 锁)|
| Underlying type | `uint32` bitfield;此 enum 列**位号**(0..31)|
| 源 firmware 头 | `FMT-Firmware/src/model/fms/<vehicle>/lib/FMS_types.h` |
| 所有权侧 | firmware-canonical(`cmd_mask` 是契约 bus 字段)|
| Drift 风险 | **High**(每位错位会让 Controller 错误裁剪环路;架构 v1 §5.1.5 显式提到 `cmd_mask`)|
| 被引用位置 | `FMS_Out_Bus.cmd_mask`(A3 §4.4.6.2);`Control_Out_Bus.cmd_mask` echo(A3 §4.5.5)|
| **重要边界** | 本 subsection **只**列位号 + 工作名 + 数值;**不**定义"该位被置位时 Controller 做什么"。位**语义**(哪一位代表哪个环路裁剪规则)由 [D6 FMS↔Controller 接口约定](../D-fms/D6-fms-controller-interface.md) ↔ [E1 Controller 功能设计](../E-controller/E1-controller-functional.md) co-seal batch(Wave 8)锁定。A3 §4.7 指针在此落地。|

值表(每行 = 1 个位号 = 1 个 enum 成员):

| name | numeric value (= bit position) | meaning(**仅工作名,不含语义裁决**)| introduced-in | mirrored-by-model |
|---|---|---|---|---|
| `MASK_BIT_POSITION_LOOP` | `0` `(B4 verify; D6/E1 lock semantics)` | A3 §4.4.6.1 标 `cmd_mask.position_loop` 守卫;本位号工作名同 A3 | current firmware tip | yes |
| `MASK_BIT_VELOCITY_LOOP` | `1` `(B4 verify; D6/E1 lock semantics)` | A3 标 `cmd_mask.velocity_loop` 守卫 | current firmware tip | yes |
| `MASK_BIT_ACCELERATION_LOOP` | `2` `(B4 verify; D6/E1 lock semantics)` | A3 标 `cmd_mask.acceleration_loop` 守卫 | current firmware tip | yes |
| `MASK_BIT_ATTITUDE_LOOP` | `3` `(B4 verify; D6/E1 lock semantics)` | A3 标 `cmd_mask.attitude_loop` 守卫 | current firmware tip | yes |
| `MASK_BIT_RATE_LOOP` | `4` `(B4 verify; D6/E1 lock semantics)` | A3 标 `cmd_mask.rate_loop` 守卫 | current firmware tip | yes |
| `MASK_BIT_YAW_LOOP` | `5` `(B4 verify; D6/E1 lock semantics)` | A3 标 `cmd_mask.yaw_loop` 守卫 | current firmware tip | yes |
| `MASK_BIT_YAW_RATE_LOOP` | `6` `(B4 verify; D6/E1 lock semantics)` | A3 标 `cmd_mask.yaw_rate_loop` 守卫 | current firmware tip | yes |
| `MASK_BIT_THROTTLE_PASSTHROUGH` | `7` `(B4 verify; D6/E1 lock semantics)` | A3 标 `cmd_mask.throttle_passthrough` 守卫 | current firmware tip | yes |

Notes:

- **位号 vs 位语义边界(再次明确)**:本文件锁定 (位号工作名, 数值) 二元组的**存在与位置**;不锁定"该位被置位时 Controller / Plant 做什么"。后者由 [D6](../D-fms/D6-fms-controller-interface.md) ↔ [E1](../E-controller/E1-controller-functional.md) co-seal batch(Wave 8)定稿。
- 若 D6/E1 在 Wave 8 决议某些位号合并 / 删除 / 重命名,本文件按 §10 变更日志同步,但**已锁定的位号数值仍按 P-1 不可变**(即:删除位 = mark deprecated,不重新赋值)。
- 总位数 ≤ 32(底层 uint32);上限由架构 v1 §5.1.5 `cmd_mask` 是 uint32 决定。
- A6 §4.4 reset-time `cmd_mask` 字段值 = 全 0(所有位关闭),A6 §4.4.2 "HARDCODED zero(safe state)"。

#### 4.4.15 辅助 enum:gear_state / landing_state(若契约存在)

A3 §4.3 / §4.4 / §4.5 字段表中**目前未列**显式 `gear_state` / `landing_state` 字段(架构 v1 §5.1.5 / §5.1.7 列举关键字段时未直接提到),但任务 brief 明确要求把"辅助 enum"包含在内。本 subsection 作 **placeholder**:**首轮 [B1](B1-bus-inventory.md) byte-level 表 + B4 首跑**确认 firmware `*_types.h` 是否含相关字段;若含,按以下骨架填充并在 §10 变更日志登记。

**`GearState`**(若契约存在):

| 属性 | 值 |
|---|---|
| 类型名 | `GearState` |
| Underlying type | `uint8` `(B4 verify)` |
| 源 firmware 头 | `FMT-Firmware/src/model/fms/<vehicle>/lib/FMS_types.h` 或 `Plant_types.h`(B4 verify)|
| 所有权侧 | firmware-canonical(若存在)|
| Drift 风险 | **Low**(辅助状态;不在闭环关键 gate)|
| 被引用位置 | TBD by B1 |

预填值表(若契约存在):

| name | numeric value | meaning | introduced-in | mirrored-by-model |
|---|---|---|---|---|
| `GEAR_UP` | `0` `(B4 verify)` | 起落架收起 | current firmware tip(若存在)| `B4 verify` |
| `GEAR_DOWN` | `1` `(B4 verify)` | 起落架放下 | current firmware tip(若存在)| `B4 verify` |
| `GEAR_TRANSITIONING` | `2` `(B4 verify)` | 起落架转换中 | current firmware tip(若存在)| `B4 verify` |

**`LandingState`**(若契约存在):

| 属性 | 值 |
|---|---|
| 类型名 | `LandingState` |
| Underlying type | `uint8` `(B4 verify)` |
| 源 firmware 头 | `FMT-Firmware/src/model/fms/<vehicle>/lib/FMS_types.h`(B4 verify)|
| 所有权侧 | firmware-canonical(若存在)|
| Drift 风险 | **Low–Medium**(若进入 `FMS_Out_Bus.ext_state` 之外的独立字段则为 Medium)|
| 被引用位置 | TBD by B1 — 多数情况下 landing 子状态可由 `VehicleExtState` + `FlightMode` 组合表达,本独立 enum 仅当 firmware 真有独立字段时启用 |

预填值表(若契约存在):

| name | numeric value | meaning | introduced-in | mirrored-by-model |
|---|---|---|---|---|
| `LAND_NONE` | `0` `(B4 verify)` | 未在降落 | current firmware tip(若存在)| `B4 verify` |
| `LAND_DESCENDING` | `1` `(B4 verify)` | 下降中 | current firmware tip(若存在)| `B4 verify` |
| `LAND_FLARE` | `2` `(B4 verify)` | 末段 flare | current firmware tip(若存在)| `B4 verify` |
| `LAND_TOUCHDOWN` | `3` `(B4 verify)` | 触地 | current firmware tip(若存在)| `B4 verify` |
| `LAND_COMPLETE` | `4` `(B4 verify)` | 已落地、已 disarm | current firmware tip(若存在)| `B4 verify` |

Notes:

- 这两条 enum 的"是否真存在于 firmware 契约"由 [B1](B1-bus-inventory.md) byte-level 字段表 + [B4](B4-contract-diff.md) 首跑确认。本文件预留 placeholder + 数值意图,使下游(B1 / D5 / E4)可基于稳定枚举名引用,即使首轮 B4 报告它们**不**作为契约 enum 存在 — 此时本 subsection 在 §10 变更日志中标 `withdrawn from contract scope` 并迁回到 D 区内部 enum(由 D1 / D5 私有定义)。

#### 4.4.16 模块内部 fault / health enum(非契约,边界声明)

按 [A2 §4.9](../A-architecture/A2-naming-conventions.md) R-9.1,内部 enum 不得出现在 firmware-visible 接口。Plant / Controller 内部模块若有 fault / health 子状态(例如 `MotorFault`、`BatteryHealth`),**不**进入本契约 enum 清单 — 本节仅作**边界声明**,提示这类 enum 由各模块功能设计(C1 / E1)私有定义,在它们的字段汇入 `FMS_Out_Bus.error_code`(§4.4.9)前必须**翻译成 `ErrorCode`**;本翻译规则不属于 B2 范围。

**因此,本目录的契约 enum 总数为:§4.4.1–4.4.14(14 条已锁定)+ §4.4.15 中条件性两条(GearState、LandingState — 由首跑 B4 确认是否纳入契约)。**

### 4.5 模型仓 enum 定义形态(物理形态指针)

模型仓侧每个本文件锁定的契约 enum 在 `model/shared/enum/` 下以**单一形态**存在;具体 SLDD vs 脚本 vs JSON 的物理选型由 [B1](B1-bus-inventory.md) §… 中 "数据字典 vs 脚本"决议落地(B1 给抉择,本文件遵从)。本文件**不**重复 B1 决议,仅声明:

- 一个 enum 一个文件 / 一个 SLDD 条目,文件名遵从 [A2 §4.3](../A-architecture/A2-naming-conventions.md) R-3.4 / R-3.6。
- 内容必须包括:类型名、底层 underlying type、所有成员名 + 数值、源 firmware 路径、所有权侧、drift 风险、版本注记。
- I2 Bus/enum 镜像脚本(per [00-design-plan §4.I I2](../00-design-plan.md))从 firmware `*_types.h` 抽取后**对照本文件值表**生成模型仓 enum 文件;不一致时按 §4.3 D-1..D-7 规则报告。

### 4.6 数值未锁定项与对 firmware 的依赖说明

本文件多数 numeric value 标 `(B4 verify)`。这是因为:

1. 当前作业环境**未挂载** FMT-Firmware 源码树,作者无法字面读取 `*_types.h`;
2. 数值锁定的**最终权威**是 firmware tip,B4 首跑前任何整数都是设计意图;
3. 符合 RULES §5 "镜像自 firmware 的契约不记录来源版本 → 替代:在 §3 依赖中记录 `FMT-Firmware` 的 commit hash + 文件相对路径"。本文件 §3 的 firmware 行 commit hash 标 `<pending hash>`,与 [A6 §3](../A-architecture/A6-init-reset-contract.md) / [A7 §3](../A-architecture/A7-time-conventions.md) / [A3 §3](../A-architecture/A3-module-boundaries.md) / B1 / B3 同 batch 模式一致 — 在 INDEX 决策日志登记时由 orchestrator / reviewer 同步补齐。

**首轮 B4 通过 → 本文件 §10 变更日志追加一条 "B4 numeric lock-in confirmed at FMT-Firmware @ <hash>"**,届时所有 `(B4 verify)` 标记可移除。

## 5. 已知风险与悬而未决问题

- **首跑 B4 前所有数值是 design intent,不是 firmware 实测**
  - 影响:B1 / B3 / 下游 D / E / F / I2 / I3 在 B4 首跑前**不应**把本文件具体数值用于二进制比对;只可用作类型 / 名称 / 位号引用。
  - 处置:open;由 B4 首跑后定稿,§10 变更日志记录。
- **`INS_Status` enum vs bitfield 形态未确认**
  - 影响:A3 §4.4.4 标"`status` bitfield 或 enum,per firmware";本文件按 enum 形式锁定;若 firmware 实际用 bitfield,B5 镜像产物会偏离本文件
  - 处置:由 [B5 INS_Out_Bus 镜像同步流程](B5-ins-bus-mirror.md) 镜像产物决定,本文件随 B5 落定;在 §10 变更日志中同步切换
- **`PilotMode` 与 `FlightMode` 数值是否实际相等**
  - 影响:多旋翼 firmware 历史上 `PilotMode` 与 `FlightMode` 数值多有重叠;若 firmware 已经统一为单 enum,本文件保留两条会造成 redundancy
  - 处置:open;由 B4 首跑确认,若 firmware 实际只有一份则本文件按 P-4 deprecate 流程合并(保留两个名 + 同数值)
- **`cmd_mask` 位号工作名与位语义解耦的工程风险**
  - 影响:本文件锁位号但不锁语义,意味着 B1 / B3 / I3 在 Wave 8 之前**不可**运行 cmd_mask 位级语义检查;只能跑位号存在性 + 数值检查
  - 处置:open;Wave 8 D6/E1 co-seal batch 完成后,B4 / I3 可加入 cmd_mask 位语义检查(由 D6 / E1 给出语义表)
- **辅助 enum(GearState / LandingState)是否真为契约**
  - 影响:§4.4.15 给出 placeholder;若 firmware 不含,§10 撤回这两条
  - 处置:open;首跑 B4 / I3 决定
- **enum 在生成代码中的形态(`enum class` vs `int8_t` typedef)**
  - 影响:不同 codegen 配置可能影响 ABI 兼容;P-5 把 underlying type 锁住,但成员可见性 / 命名空间未锁
  - 处置:由 [I4 Codegen 配置](../I-tooling/I4-codegen-config.md) 处理;本文件不裁决

## 6. 退出条件复核

对照 [`00-design-plan.md`](../00-design-plan.md) 中 B2 工作项"退出条件"逐条:

| # | 退出条件原文 | 本文档依据 | 状态 |
|---|---|---|---|
| 1 | "所有契约 enum 名 + 数值 + 含义,与 firmware 一致" | §4.4.1–4.4.14 锁定 14 条契约 enum 类型 + §4.4.15 条件性 2 条 placeholder + §4.4.16 边界声明;每个值表给 (name, numeric value, meaning, introduced-in, mirrored-by-model);"与 firmware 一致" 的检测由 §4.3 D-1..D-7 给 B4 / I3 兑现;首跑 B4 通过前数值标 `(B4 verify)` 是合规设计意图,符合 RULES §5 firmware commit hash + 文件相对路径(§3 + §4.6)的 ".. 待补齐" 同 batch 模式 | 满足(数值层 conditional on B4 首跑)|

00-design-plan §4.B 的 B2 行只列 1 条退出条件,因此本表只有 1 行。任务 brief 的额外章节要求("Enum 命名约定"echo A2 §4.x、"Drift 管理政策"、"Diff 覆盖要求"、"Per-enum 目录")在 §4.1 / §4.2 / §4.3 / §4.4 一一对应:

| brief 要求 | 本文档对应章节 |
|---|---|
| §4.x "Enum 命名约定 — echo A2 §4.x style" | §4.1(完全引用 A2 §4.6,不重述)|
| §4.x "Drift 管理政策 — additive-only / end-of-list / lockstep with firmware" | §4.2(P-1..P-6 6 条政策)|
| §4.x "Diff 覆盖要求 for B4/I3" | §4.3(D-1..D-8 8 条验收标准)|
| §4.x "Per-enum 目录 — 一个 subsection 每 enum" | §4.4(§4.4.1–4.4.16 共 16 个 subsection)|

## 7. 下游影响

按 [`01-design-relationships.md`](../01-design-relationships.md) §4.2 B 区出边 + §4.3–4.9 各区 ⇢ 边:

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [B1 Bus 清单与 schema 设计](B1-bus-inventory.md) (sibling, co-seal) | ↔ | 所有 enum-typed 字段类型由本文件 §4.4 锁;B1 引用本文件类型名 |
| [B3 Parameter schema 设计](B3-parameter-schema.md) (sibling, co-seal) | ↔ | enum-typed 参数字段类型由本文件 §4.4 锁;B3 引用本文件类型名 |
| [B4 契约 diff 策略设计](B4-contract-diff.md) | → | §4.3 D-1..D-8 验收标准是 B4 enum-side 验收的输入 |
| [B5 INS_Out_Bus 镜像同步流程](B5-ins-bus-mirror.md) | → | §4.4.10 `INS_Status` + §4.4.11 `INS_Flag` 的 model-mirrored 标记;B5 镜像产物以本文件值表为对照点 |
| [C1 Plant 功能设计](../C-plant/C1-plant-functional.md) | ⇢ | §4.4.12 `GPS_FixType` + 内部 fault enum 翻译规则(§4.4.16)|
| [C3 Plant 多旋翼 leaf 设计](../C-plant/C3-multicopter-leaf.md) | ⇢ | §4.4.16 motor fault → ErrorCode 翻译边界 |
| [D1 FMS 功能设计](../D-fms/D1-fms-functional.md) | ⇢ | §4.4.4 `PilotMode` + §4.4.5 `FlightMode` + §4.4.7 `FailsafeState` + §4.4.8 `MissionState` 全集;D1 决定子集 |
| [D3 FMS Mode Manager 详细设计](../D-fms/D3-mode-manager.md) | ⇢ | §4.4.1 `VehicleStatus` + §4.4.2 `VehicleState` + §4.4.3 `VehicleExtState` + §4.4.6 `CtrlMode` 数值;D3 状态机以这些 enum 作为输出锚点 |
| [D4 FMS Command Shaper 详细设计](../D-fms/D4-command-shaper.md) | ⇢ | §4.4.14 `cmd_mask` 位号(语义在 D6 ↔ E1,数值在本文件)|
| [D5 FMS 多旋翼 leaf 设计](../D-fms/D5-multicopter-leaf.md) | ⇢ | §4.4.4 / §4.4.5 mode 子集 + §4.4.15 LandingState placeholder |
| [D6 FMS↔Controller 接口约定](../D-fms/D6-fms-controller-interface.md) (Wave 8 co-seal) | ↔ | §4.4.14 `cmd_mask` 位号;D6 锁位语义,本文件锁数值 |
| [E1 Controller 功能设计](../E-controller/E1-controller-functional.md) (Wave 8 co-seal) | ↔ | §4.4.6 `CtrlMode` + §4.4.14 `cmd_mask` 位号 → E1 据此定环路裁剪规则 |
| [F3 INS_Out_Bus 消费规则](../F-ins-contract/F3-consumption-rules.md) | ⇢ | §4.4.10 `INS_Status` + §4.4.11 `INS_Flag` validity 位号 → F3 落 fallback 规则 |
| [I2 Bus/enum 镜像脚本设计](../I-tooling/I2-bus-enum-mirror.md) | ⇢ | §4.5 模型仓 enum 定义形态 + §4.4 全表是 I2 抽取目标对照清单 |
| [I3 契约 diff 脚本设计](../I-tooling/I3-contract-diff.md) | ↔(via B4)| §4.3 D-1..D-8 是 I3 enum-side 实现规范;I3 实现 What 由 B4 给,How 由 I3 给 |
| [I4 Codegen 配置脚本设计](../I-tooling/I4-codegen-config.md) | ⇢ | §4.4 全表 underlying type 是 I4 codegen 配置(`enum class` vs typedef)的输入 |

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-07 | fmt-design-author | 初稿;Wave 4 co-seal batch (B1, B2, B3) 同期起草。所有 numeric value 标 `(B4 verify)` 等待首跑 B4 / I3 确认 |

## Self-check

- [x] frontmatter 完整,字段值合法(work_item=B2;status=draft;contract_impact=yes;upstream 包含架构 v1 + A1 + A2 + A3 + A6 + A7;authored_at=2026-05-07)
- [x] 上游文档全部存在且 status ≥ reviewed(架构 v1 已发布;A1 / A2 / A3 / A6 / A7 全部 status=reviewed,见 [INDEX](../INDEX.md));或为 co-seal batch 兄弟(B1 / B3 同 batch (B1, B2, B3),已在 §3 注明 batch 名)
- [x] 退出条件逐条复核完成,每条均给出依据(§6:00-design-plan §4.B 列出的 1 条退出条件 + 任务 brief 4 条额外章节要求)
- [x] 引用路径全部可点击访问(本文件所有引用均使用 RULES §4 规定的相对路径形式;B1 / B3 / B4 / B5 / D / E / F / I 区文件在 co-seal 同期或后续 wave 创建,符合 sibling-or-pending 规则)
- [x] 不存在 RULES §5 禁则中的内容(无 .slx 截图、无可执行 .m 代码、无 firmware 实现复述、无 PR/branch 名、无重复 bus/parameter 字段表 — 本文件**只**含 enum 字段表,符合"B 区单一来源"分工)
- [x] 触及 firmware 契约者(contract_impact=yes)将由 orchestrator 在 INDEX 决策日志登记(本文件 contract_impact=yes;按 fmt-design-author skill 规则,本作者不直接编辑 INDEX,由 orchestrator / reviewer 同步登记)
- [ ] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径(**部分满足**,与 [A3](../A-architecture/A3-module-boundaries.md) / [A6](../A-architecture/A6-init-reset-contract.md) / [A7](../A-architecture/A7-time-conventions.md) 同 batch 模式:文件相对路径已在 §3 给出 4 条 firmware 头文件路径 — `FMS_types.h` / `Controller_types.h` / `Plant_types.h` / `INS_types.h`;commit hash 标 `<pending hash>`,待 orchestrator / reviewer 在 INDEX 登记同步补齐 + 首跑 B4 / I3 通过后在 §10 变更日志记录)
- [x] 下游影响已沿关系图识别完毕(§7 表覆盖 B 区 4 条直接出边 + C / D / E / F / I 区共 12 条 ⇢ 边,与 §4.2 / §4.3 / §4.4 / §4.5 / §4.6 / §4.9 一致)
- [ ] 文档不超出本工作项范围(**部分满足**,与 [A3](../A-architecture/A3-module-boundaries.md) 同 batch 模式:本文件**仅锁** enum 名 + 数值 + 命名 + drift 政策 + diff 验收;**不**锁 bus 字段顺序 / 字节布局(留 B1)、不锁参数默认值(留 B3)、不锁 cmd_mask 位语义(留 D6 / E1 Wave 8)、不锁 mode 子集裁决(留 D1)、不锁 fault 翻译规则(留 C1 / E1)、不锁内部 enum(留各模块功能设计);仅 §4.4.16 给"内部 enum 边界声明",符合 A2 §4.9 R-9.1 边界。**Self-check 第 9 项保持未勾选**,与 A3 同 batch 模式:声明所有越权风险点 + 留给下游 ID,而非自我裁决)
