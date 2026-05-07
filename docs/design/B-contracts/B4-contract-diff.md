---
work_item: B4
title: 契约 diff 策略设计
upstream: ["架构 v1", "A1", "A2", "A3", "A6", "B1", "B2", "B3"]
contract_impact: yes
status: draft
authored_at: 2026-05-07
last_reviewed_at:
reviewer_verdict: none
---

# B4 契约 diff 策略设计

## 1. 目的

为 FMT-Model-2025b 的 firmware-visible 契约(bus / enum / `*_PARAM` / `*_EXPORT` / 入口符号)定义**唯一的契约 diff 策略**:**测什么、何时测、过/不过判据、失败如何响应、报告怎样产出**,以闭合架构 v1 §17 Risk 1 (contract drift) / §17 Risk 6 (parameter governance) / §10 silent enum drift,并把 [B1 字节级 schema](B1-bus-inventory.md) / [B2 enum 数值表](B2-enum-inventory.md) / [B3 PARAM/EXPORT 字段表](B3-parameter-schema.md) 三件套定义的"模型仓侧契约设计意图"与 firmware tip 之间设立 **post-codegen guardrail**。本文件是 WHAT(策略),其 HOW(实现)由 co-seal 兄弟 [I3 契约 diff 脚本设计](../I-tooling/I3-contract-diff.md) 定义。

## 2. 范围

**在范围:**

- **§4.1 Coverage matrix**:6 条 audit-fortified 覆盖 clauses (a) 全部 bus 字段顺序+Simulink→C 类型映射+累计字节宽度等于 firmware `*_types.h`(B1 16 个 bus);(b) 全部 enum 名+数值精确等于 firmware(B2 §4.4.x);(c) `FMS_PARAM`/`CONTROL_PARAM`/`PLANT_PARAM` 字段顺序+类型等于 firmware(B3 §4.3/§4.4/§4.5);(d) `FMS_EXPORT`/`CONTROL_EXPORT`/`PLANT_EXPORT` 字段顺序+`period` 数值+`model_info[]` 长度(B3 §4.7);(e) 入口符号存在性 — `FMS_init`/`FMS_step`/`Controller_init`/`Controller_step`/`Plant_init`/`Plant_step` 必须出现在 codegen 产物;(f) PARAM 全局符号存在性 — `FMS_PARAM`/`CONTROL_PARAM`/`PLANT_PARAM` 全局名必须出现在 codegen 产物
- **§4.2 Pass / fail thresholds**:每行覆盖矩阵的 severity (`fail` / `warn`)、tolerance、acceptance criterion
- **§4.3 Run timing**:pre-export / post-export / pre-merge gate / pre-release gate 四套触发点的覆盖差异
- **§4.4 Failure-response policy**:HARD-FAIL / WARN / 三角化路径(谁 review、看什么 artifact、何时可用 `--allow-drift` 与审计痕迹)
- **§4.5 Report format**:JSON schema 与 Markdown 表格式(I3 兑现);worked example fragment
- **§4.6 Cross-reference map to I3**:每条 B4 检查 → I3 §x.x 章节 → 数据源
- **§4.7 Lockstep policy**(per 架构 v1 §15):模型仓 PR 触及 B1/B2/B3 必跑 B4,firmware-tip 漂移检出触发 INDEX 决策日志

**不在范围(由其他工作项处理):**

- 脚本实现 / CLI / 入参 / 解析逻辑 — 由 [I3 契约 diff 脚本设计](../I-tooling/I3-contract-diff.md) 处理(co-seal 兄弟)
- bus 字段 / enum 数值 / PARAM 字段表本身 — 已在 [B1](B1-bus-inventory.md) / [B2](B2-enum-inventory.md) / [B3](B3-parameter-schema.md) 定义,本文件只引用并定义 diff 规则
- `INS_Out_Bus` 镜像产物的 canonical-source 流程 — 由 [B5 INS_Out_Bus 镜像同步流程](B5-ins-bus-mirror.md) 处理;B4 把 `INS_Out_Bus` 视作 B1 §4.3.1 中的一条 bus 一并 diff,不区分镜像方向
- bus / enum 头文件抽取脚本 — 由 [I2 Bus/enum 镜像脚本设计](../I-tooling/I2-bus-enum-mirror.md) 处理;B4 / I3 与 I2 的 YAML 镜像产物有可选依赖(per §4.6),但 B4 策略层只声明输入接口,不规定 I2 输出格式
- ert.tlc codegen 配置项 — 由 [I4 Codegen 配置脚本设计](../I-tooling/I4-codegen-config.md) 处理;B4 在 codegen 产物上做 post-export check,但不规定如何配置 codegen
- `cmd_mask` 位号语义 / `INS_Status` / `INS_Flag` 位 alias 的具体位号 — 由 [D6](../D-fms/D6-fms-controller-interface.md) + [E1](../E-controller/E1-controller-functional.md) co-seal batch (Wave 8) 锁定;B4 只把这些 enum 视为 B2 §4.4.x 条目并按 (b) 规则 diff
- 模型仓 SIL/SIH/HIL 行为对比 — 不属契约 diff 范围

## 3. 依赖

| 上游 | 引用位置 | 用途 |
|---|---|---|
| 架构 v1 §5.1.1 / §5.1.2 / §5.1.7 / §5.1.8 / §10 / §13 / §14.4 / §15 / §17 Risk 1 / Risk 6 | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | 契约不可变原则 / `*_init` `*_step` `*_PARAM` `*_EXPORT` 符号必现 / silent enum drift / `period` 数值表 / contract review gates / lockstep / 风险表 |
| [A1 架构 v1 评审与缺口闭合](../A-architecture/A1-v1-review.md) (status: reviewed, 2026-05-05) | A1 §5.1.5 / §5.1.7 / §5.1.9 / §17 Risk 6 | 字段顺序 / enum drift / parameter governance 三个 critical 缺口要求由 B4 + I3 联合闭合 |
| [A2 命名与目录约定设计](../A-architecture/A2-naming-conventions.md) (status: reviewed, 2026-05-05) | A2 §4.6 R-6.3 / §4.7 R-7.x / §4.8 R-8.1 | enum 数值锁定;PARAM struct 名 `<MODULE>_PARAM`;顶层模型根名 → 入口符号名映射 |
| [A3 模块边界信号清单细化](../A-architecture/A3-module-boundaries.md) (status: reviewed, 2026-05-07) | A3 §4.3..§4.9 字段元数据;§4.12 元数据要求 | 16 个 contract bus 的字段集合上限(B4 验证 firmware tip 与该集合等价) |
| [A6 Init/Reset 状态契约](../A-architecture/A6-init-reset-contract.md) (status: reviewed, 2026-05-05) | A6 §4.2 init 行为契约 | 入口符号 `*_init` / `*_step` 是 init 契约的可观测面;符号丢失 = init 契约破坏 |
| [B1 Bus 清单与 schema 设计](B1-bus-inventory.md) (status: reviewed, 2026-05-07) | B1 §4.1 endianness / 类型映射 / 累计 offset 约定;§4.2..§4.8 16 bus 字段表;§4.9 cross-ref;§4.10 与 B4 退出条件呼应表;§4.11 rate;§4.12 16 / 16 全覆盖 | (a) 覆盖项的字节级 baseline(模型仓侧设计意图);diff 工具的"模型仓侧契约"输入 |
| [B2 Enum 清单与数值锁定](B2-enum-inventory.md) (status: reviewed, 2026-05-07) | B2 §4.2 drift policy P-1..P-6;§4.4.1..§4.4.16 enum 数值表 | (b) 覆盖项的 enum 名+数值 baseline;drift 政策 P-2 末尾追加规则 → B4 §4.2 "warn vs fail" 判据 |
| [B3 Parameter schema 设计](B3-parameter-schema.md) (status: reviewed, 2026-05-07) | B3 §4.3 / §4.4 / §4.5 PARAM 字段表;§4.7 EXPORT struct + model_info[] ledger;§4.8 diff coverage 要求 | (c) (d) 覆盖项的 baseline;`period` 数值(Plant=1ms / Controller=5ms / FMS=20ms,per 架构 v1 §13)|
| **co-seal batch (B4, I3)**:[I3 契约 diff 脚本设计](../I-tooling/I3-contract-diff.md) (sibling, draft) | per [01-design-relationships.md §5 协同对 3](../01-design-relationships.md) — entry 3 = `(B4, I3)` | **本工作项与 I3 同 batch(Wave 5 协同定稿)。B4 owns WHAT (策略);I3 owns HOW (实现)。本文件 §4.6 列出每条 B4 检查 → I3 实现章节的映射;允许引用 I3 draft per RULES §6 self-check 第 2 项(co-seal 同批互引)** |
| firmware `*_types.h` 与 codegen 产物(契约 diff 的另一端) | `FMT-Firmware/src/model/plant/<vehicle>/lib/Plant_types.h`;`FMT-Firmware/src/model/fms/<vehicle>/lib/FMS_types.h`;`FMT-Firmware/src/model/control/<vehicle>/lib/Controller_types.h`;`FMT-Firmware/src/model/ins/lib/INS_types.h`;以及包含 `*_PARAM` / `*_EXPORT` 全局声明的相关头文件;**FMT-Firmware @ \<pending hash\>**(commit hash 由 INDEX 决策日志维护方在登记本工作项时补齐,与 [B1 §3 / B2 §3 / B3 §3](B1-bus-inventory.md) 同样的占位约定;FMT-Firmware 仓未挂载在本环境)| diff 工具的"firmware 端真值"输入。本工作项不复述这些头文件内容;只声明它们是 diff 的另一端 |

注:**co-seal batch 名 = (B4, I3)**(per [01-design-relationships.md §5 协同对 3](../01-design-relationships.md) entry 3)。本文件作为 B4,在 §3 引用 I3 兄弟 draft 是 RULES §6 self-check 第 2 项允许的,无须等 I3 进入 reviewed。

环境注:FMT-Firmware 仓在本设计阶段未挂载,本文件不指定具体 firmware 字段值。所有 firmware 路径用 `FMT-Firmware @ <pending hash>` 占位;实际 firmware tip 数据由 I3 在 codegen 导出阶段读取。本文件 contract_impact=yes 因为 diff 策略一旦发布,任何放宽 / 收紧条款都直接影响 firmware-visible 契约保护范围。

## 4. 设计内容

### 4.1 Coverage matrix(契约 diff 必跑全表)

下表是契约 diff 工具(由 [I3](../I-tooling/I3-contract-diff.md) 实现)**必须**跑的全部检查,按 6 个 audit-fortified clauses (a)..(f) 组织。每行有唯一 `Check ID` 供 I3 在报告中回引(per §4.5 / §4.6)。

每条 Check ID 命名规则:`B4-<clause-letter><nn>` — 例 `B4-a01`(clause (a) 第 1 项)。

| Check ID | Clause | 检查对象 | 来源(模型仓侧) | 来源(firmware 端) | 检查内容 |
|---|---|---|---|---|---|
| B4-a01 | (a) bus 字段顺序 | 16 个 contract bus(`Pilot_Cmd_Bus` / `GCS_Cmd_Bus` / `Auto_Cmd_Bus` / `Mission_Data_Bus` / `INS_Out_Bus` / `FMS_Out_Bus` / `Control_Out_Bus` / `Plant_States_Bus` / `Extended_States_Bus` / `Environment_Info_Bus` / `States_Init_Bus` / `IMU_Bus` / `MAG_Bus` / `Barometer_Bus` / `GPS_uBlox_Bus` / `AirSpeed_Bus`)| [B1 §4.2..§4.8](B1-bus-inventory.md) 字段表 Order 列 | `*_types.h` 中 `typedef struct ... <Bus>;` 字段声明顺序 | 字段名集合 + 字段顺序逐字段相等 |
| B4-a02 | (a) bus 字段 Simulink→C 类型映射 | 同上 16 bus | B1 §4.2..§4.8 Type 列 + B1 §4.1.2 类型映射表(`single`→`real32_T`、`uint8`→`uint8_T` 等)| 同上 firmware `*_types.h` 字段类型 | 每字段:模型仓 Simulink 类型经 §4.1.2 映射后的 C 类型 == firmware C 类型(包括数组维度、嵌套 struct 名)|
| B4-a03 | (a) bus 累计字节宽度 | 同上 16 bus | B1 §4.2..§4.8 Width 列 + 各 bus "Total bus byte width" 行 | firmware `sizeof(<Bus>)` 编译期常量(可由 I3 静态计算 padding) | 每 bus 模型仓累计宽度 == firmware sizeof,**含 alignment padding**(per B1 §4.1.3 natural alignment 假设)|
| B4-a04 | (a) bus endianness | 同上 16 bus(全局)| B1 §4.1.1 little-endian 声明 | firmware 编译目标 ARM Cortex-M(little-endian) | 编译目标字节序 == little-endian(对 firmware ELF / target triple 静态检查)|
| B4-a05 | (a) 嵌套子 schema | `quat[4]` / `pos_ned[3]` / `vel_ned[3]` / `ang_rate_b[3]` / `euler[3]` / `INS_Status` / `INS_Flag` / `Waypoint`(B1 §4.6)| B1 §4.6 子 schema | firmware `*_types.h` 中对应 typedef | 子 schema 字段顺序 + 类型 + 总宽度等于 firmware |
| B4-b01 | (b) enum 类型存在性 | B2 §4.4.1..§4.4.16 全部契约 enum 类型(`VehicleStatus` / `VehicleState` / `VehicleExtState` / `PilotMode` / `FlightMode` / `CtrlMode` / `FailsafeState` / `MissionState` / `ErrorCode` / `INS_Status` / `INS_Flag` / `GPS_FixType` / `GCS_CmdType` / `cmd_mask` 位号 / `gear_state` / `landing_state` 若契约存在)| B2 §4.4.x 类型清单 | firmware `*_types.h` `typedef enum` 名 | 模型仓侧每个 enum 类型在 firmware 存在 |
| B4-b02 | (b) enum 成员名+数值精确相等 | 同上每 enum 的全部成员 | B2 §4.4.x 各 enum 值表 (name + numeric value) | firmware `*_types.h` 各 enum 成员声明 | 模型仓侧与 firmware **逐成员**:name + numeric value 精确相等(零 tolerance)|
| B4-b03 | (b) enum 成员集合双向覆盖 | 同上 | B2 §4.4.x 全清单 | firmware 全清单 | 模型仓侧成员 ⊆ firmware,**且** firmware 侧成员 ⊆ 模型仓侧 — 区分两种漂移情况见 §4.2 |
| B4-b04 | (b) enum 底层整型宽度 | 同上 | B2 §4.4.x header "underlying type" 列(default `uint8`,bitfield `uint32`)| firmware `enum` 显式或隐式底层类型(`__attribute__((packed))` / `: uint8_t`)| 底层整型宽度等于 — 任意宽度变化触发 P-5 契约破坏(per B2 §4.2)|
| B4-c01 | (c) PARAM 字段顺序 | 3 个 PARAM struct(`PLANT_PARAM` / `FMS_PARAM` / `CONTROL_PARAM`)| [B3 §4.3.2 / §4.4.2 / §4.5.2](B3-parameter-schema.md) 字段表 `#` 列顺序 | firmware `*_types.h` 中 `typedef struct ... <MODULE>_PARAM_TYPE;` 字段声明顺序 | 字段顺序逐字段相等 |
| B4-c02 | (c) PARAM 字段类型 | 同 3 个 PARAM struct | B3 字段表 Type / Width 列 | firmware 字段类型 | 每字段 Simulink→C 类型映射(per B1 §4.1.2)等于 firmware,包括数组长度与嵌套 struct |
| B4-c03 | (c) PARAM 字段 storage class 一致性 | 同 3 个 PARAM struct | B3 字段表 Class 列(R-T / C-I)| codegen 产物中 R-T 字段以 `extern volatile` 形式可寻址,C-I 字段不在 PARAM struct 中 | 每字段 storage class 与 codegen 配置(由 I4)+ B3 Class 列声明匹配。**注**:本检查对 firmware 不做对称要求(firmware 端未必标 storage class);只验证 model-side codegen 产出与 B3 声明一致 |
| B4-c04 | (c) PARAM 总字节宽度 | 同 3 个 PARAM struct | B3 §4.10.1 字节估算 | firmware `sizeof(<MODULE>_PARAM_TYPE)` | 估算可能因 padding 与实际不等;此检查为 `warn` 级,记录差异不阻塞 |
| B4-d01 | (d) EXPORT 字段顺序 | 3 个 EXPORT struct(`PLANT_EXPORT` / `FMS_EXPORT` / `CONTROL_EXPORT`)| [B3 §4.7.2 / §4.7.3 / §4.7.4](B3-parameter-schema.md) | firmware `*_types.h` 对应 typedef 字段声明顺序 | 字段顺序相等 |
| B4-d02 | (d) EXPORT `period` 数值 | 同 3 个 EXPORT | B3 §4.7.2..§4.7.4 默认值列(Plant=1, FMS=20, Controller=5,per 架构 v1 §13)| firmware EXPORT 实例化数值 | `period` 字段值精确相等(以 ms 为单位,uint16) |
| B4-d03 | (d) `model_info[]` 长度 | 同 3 个 EXPORT | B3 §4.7.2..§4.7.4 `char[N]` 中 N 占位 | firmware EXPORT `model_info` 数组声明长度 | `N` 精确相等 |
| B4-d04 | (d) EXPORT 总字节宽度 | 同 3 个 EXPORT | B3 §4.7 EXPORT struct sketch | firmware `sizeof(<MODULE>_EXPORT_TYPE)` | 总宽度精确相等(含 alignment padding) |
| B4-e01 | (e) 入口符号存在性 — Plant | codegen 产物 `Plant.c` / `Plant.h` 与 ELF symbol table | 不适用(模型仓侧产出物即检查目标) | 入口符号期望(per 架构 v1 §5.1.2)| `Plant_init` + `Plant_step` 同时出现在生成 `.c` 中作为函数定义;若 ELF 可用,出现在 symbol table |
| B4-e02 | (e) 入口符号存在性 — FMS | codegen 产物 `FMS.c` / `FMS.h` | 同上 | 同上 | `FMS_init` + `FMS_step` 同时存在 |
| B4-e03 | (e) 入口符号存在性 — Controller | codegen 产物 `Controller.c` / `Controller.h` | 同上 | 同上 | `Controller_init` + `Controller_step` 同时存在 |
| B4-f01 | (f) PARAM 全局符号存在性 — Plant | codegen 产物 `.c` / `.h` 与 ELF symbol table | 不适用(模型仓侧产出物即检查目标) | per 架构 v1 §5.1.2:`PLANT_PARAM` 必须作为全局可寻址符号 | `PLANT_PARAM` 全局名出现在 `.c` 中作为 `<MODULE>_PARAM_TYPE PLANT_PARAM = {...};`;ELF symbol table(若可用)有该符号 |
| B4-f02 | (f) PARAM 全局符号存在性 — FMS | 同上 | 同上 | 同上(`FMS_PARAM`)| `FMS_PARAM` 全局名存在 |
| B4-f03 | (f) PARAM 全局符号存在性 — Controller | 同上 | 同上 | 同上(`CONTROL_PARAM`)| `CONTROL_PARAM` 全局名存在 |

**总检查项数:24** 项(a:5, b:4, c:4, d:4, e:3, f:3)。

**矩阵覆盖证明**(对照任务 brief 与 [00-design-plan §4.B B4 退出条件](../00-design-plan.md)):

| audit-fortified clause | 覆盖范围 | Check ID 集 | 状态 |
|---|---|---|---|
| (a) 全部 bus 字段顺序+类型映射+累计字节宽度 | B1 §4.2..§4.8 16 / 16 bus + 子 schema + endianness | B4-a01..a05 | 满足 |
| (b) 全部 enum 名+数值精确相等 | B2 §4.4.1..§4.4.16 全清单 + 底层宽度 | B4-b01..b04 | 满足 |
| (c) `*_PARAM` 字段顺序+类型 | B3 §4.3.2 / §4.4.2 / §4.5.2 三个 struct + storage class 一致性 + 总字节宽度 | B4-c01..c04 | 满足 |
| (d) `*_EXPORT` 字段顺序+`period`+`model_info[]` 长度 | B3 §4.7.2..§4.7.5 三个 EXPORT + 总字节宽度 | B4-d01..d04 | 满足 |
| (e) 入口符号存在性 | `*_init` + `*_step` × Plant / FMS / Controller | B4-e01..e03 | 满足 |
| (f) PARAM 全局符号存在性 | `*_PARAM` × Plant / FMS / Controller | B4-f01..f03 | 满足 |

### 4.2 Pass / fail thresholds

每条 Check ID 的 severity / tolerance / acceptance criterion 如下表。术语:

- **`fail`**:阻塞 export(post-export 阶段)/ 阻塞 PR 合入(pre-merge 阶段);触发 §4.4 HARD-FAIL 流程
- **`warn`**:不阻塞,但写入 `report.json` 与 `report.md`,期望人工 triage(§4.4)
- **tolerance = 0**:任何差异即触发 severity;**没有 "近似相等"概念**

| Check ID | Severity | Tolerance | Acceptance criterion |
|---|---|---|---|
| B4-a01 | `fail` | 0 | 模型仓侧 bus 字段名集合 + 顺序逐字段等于 firmware;字段缺失或顺序差异 → fail |
| B4-a02 | `fail` | 0 | 每字段 Simulink→C 类型映射后等于 firmware C 类型(byte-exact 等价);包括数组长度、嵌套 struct typedef 名 |
| B4-a03 | `fail` | 0 | 每 bus `sizeof(<Bus>)` 模型仓 == firmware,byte-exact equality |
| B4-a04 | `fail` | 0 | firmware 编译目标 == little-endian;非 little-endian 即 fail(契约破坏)|
| B4-a05 | `fail` | 0 | 子 schema 字段顺序 + 类型 + 总宽度逐项等于 firmware |
| B4-b01 | `fail` | 0 | 模型仓侧每 enum 类型名在 firmware `*_types.h` 存在 |
| B4-b02 | `fail` | 0 | 每成员 (name, numeric value) 双元组与 firmware 逐成员相等;**name 大小写敏感**;**numeric value byte-exact** |
| B4-b03 | 见说明 | 0 | 模型仓多于 firmware (model-only addition) → `fail`(per B2 §4.2 P-3 lockstep 规则,模型仓不得单方面新增 enum 数值);firmware 多于模型仓 (firmware-only addition) **且** 仅在 enum 末尾追加 → `warn`(per B2 §4.2 P-2 末尾追加规则;允许 firmware 先行,模型仓在下个镜像周期同步);firmware 多于模型仓 **且** 不在末尾(中间插入或交换数值)→ `fail`(契约破坏)|
| B4-b04 | `fail` | 0 | enum 底层整型宽度等于 firmware(uint8 → uint16 等改变即契约破坏,per B2 §4.2 P-5)|
| B4-c01 | `fail` | 0 | 模型仓 PARAM 字段顺序 == firmware 字段声明顺序,name + 顺序双匹配 |
| B4-c02 | `fail` | 0 | 每字段类型(Simulink→C 映射后)与数组长度 == firmware |
| B4-c03 | `fail` | 0 | codegen 产物中字段 storage class(`extern volatile` for R-T; inline for C-I)与 B3 §4.6.4 budget 一致 |
| B4-c04 | `warn` | 0 | 模型仓估算字节宽度 vs firmware sizeof 差异 → 仅报告;不阻塞(B3 §4.10.1 已声明估算不含 padding)|
| B4-d01 | `fail` | 0 | EXPORT 字段顺序 == firmware |
| B4-d02 | `fail` | 0 | `period` 数值 byte-exact 等于 firmware(Plant=1 / Controller=5 / FMS=20,per 架构 v1 §13)|
| B4-d03 | `fail` | 0 | `model_info[]` 数组长度 N 等于 firmware |
| B4-d04 | `fail` | 0 | EXPORT 总宽度 byte-exact equality |
| B4-e01 | `fail` | 0 | ELF symbol table 含 `Plant_init` AND `Plant_step`(或 `.c` 含两者函数定义,若 ELF 不可用)|
| B4-e02 | `fail` | 0 | 同上 `FMS_init` AND `FMS_step` |
| B4-e03 | `fail` | 0 | 同上 `Controller_init` AND `Controller_step` |
| B4-f01 | `fail` | 0 | ELF symbol table 含全局 `PLANT_PARAM`(或 `.c` 含 `PLANT_PARAM_TYPE PLANT_PARAM = {...};`)|
| B4-f02 | `fail` | 0 | 同上 `FMS_PARAM` |
| B4-f03 | `fail` | 0 | 同上 `CONTROL_PARAM` |

**Severity 概览**:`fail` × 22 / `warn` × 2(B4-b03 firmware-only-additive-end-of-list 子情形 / B4-c04 PARAM 总字节宽度估算)。

**全局守则**:

1. **零 silent skip**:任一 Check ID 在某次 run 中未执行(因 firmware path missing、parse failure、tool 异常),报告必须显式标 `status = error`(I3 退出码 3,per §4.5),**不得视作 pass**。
2. **`warn` 不替代 `fail`**:任何在矩阵中标 `fail` 的检查不得通过 `--allow-drift` 在常规 export 中被静默(`--allow-drift` 触发 §4.4 审计痕迹要求)。
3. **新增检查的 severity 默认值**:本矩阵未来新增行默认 `fail`,作者必须显式标注 `warn` 才降级。

### 4.3 Run timing

下表声明 4 个触发点上每个 Check ID 是否参与:

| 触发点 | 阶段语义 | 参与的 Check IDs(集合) | 不参与的(原因)|
|---|---|---|---|
| **pre-export** | codegen 实跑前的 sanity check;只跑"轻量级"schema 比对(无需 codegen 产物)| B4-a01, a02, a04, a05; B4-b01..b04; B4-c01, c02, c04; B4-d01, d02, d03 | a03(需 firmware sizeof 静态计算,可选)/ c03(需 codegen 产物)/ d04(同)/ e01..e03 / f01..f03(全部需 codegen 产物)|
| **post-export** | codegen 完成后的全量 check;包括 codegen 产物对入口符号 / 全局 PARAM 符号的存在性扫描 | **全部 24 项**(a01..a05, b01..b04, c01..c04, d01..d04, e01..e03, f01..f03)| 无 |
| **pre-merge gate (CI)** | 模型仓 PR 上跑全量 diff;与 post-export 相同覆盖,但 CI runner 控制 firmware tip 引用为 PR base branch 上的 firmware tip | **全部 24 项** | 无 |
| **pre-release gate** | 模型仓 release tag 出仓前;firmware tip 锁定到 release SHA(由发布者声明;I3 入参)| **全部 24 项** + **快照存档**(report.json + report.md 与 firmware SHA / 模型仓 SHA 一并存档,作为 release 制品)| 无 |

**触发点之间的关系**:

```text
pre-export check ──┐
                   ├─→  post-export check  ──→  (本地 export 可成功)
                   │
                   └─→  pre-merge gate (CI) ──→  (PR 可合入)
                                                       │
                                                       ▼
                                          pre-release gate (release-tagged firmware) ──→  (release 出仓)
```

**Run timing 规则**:

1. **`pre-export` 失败 → 不允许进入 codegen**:轻量级 schema diff 已经 fail 时,跑 codegen 是浪费且可能产出错误产物。pre-export 在 export 工具链中作为 codegen 上游 stage。
2. **`post-export` 是 export 的最后一道闸门**:export 脚本(I5 或 I3 自身)在 codegen 完成后立即跑 post-export check;失败则删除/标记 codegen 产物,export 失败退出。
3. **`pre-merge gate` 在 CI 上跑,与 firmware 仓的当前 tip 比对**:本 gate 由模型仓 PR / merge 流水线触发,firmware tip 由 CI 解析 firmware 仓的当前默认分支 HEAD。任何 pre-merge gate fail 阻塞合入。
4. **`pre-release gate` 在 release branch 出仓前跑,firmware tip 锁到 release SHA**:确保发出去的 release 产物对一个明确的 firmware commit 通过 diff;`--firmware-sha` 强制由 release manager 提供。
5. **每个触发点独立,互不替代**:即使 pre-export pass,也必须跑 post-export(因为 codegen 可能引入新漂移,例如 storage class 错误配置);CI gate 不替代 release gate(因为 firmware tip 之间可能漂移)。

### 4.4 Failure-response policy

本节定义每个 Check ID 出 `fail` / `warn` / `error` 时的处理流程。**核心原则:不允许 silent skip;不允许 silent override**。

#### 4.4.1 失败级别 → 后果

| 报告 status | 触发条件 | 后果 |
|---|---|---|
| `fail` | 任一 Check ID severity=`fail` 不通过 | abort export / 阻塞 PR / 阻塞 release;退出码 2;必须产出 `report.json` + `report.md` |
| `warn` | 仅 severity=`warn` 行未通过(包括 B4-b03 firmware-additive 与 B4-c04 字节估算)| 不阻塞,但产出报告中 advisory 段落;退出码 1 |
| `error` | tool 自身故障(parse 异常、firmware path 丢失、无法读 ELF)| 不阻塞但等同 fail 阻塞效果(因为无法证明 pass);退出码 3;必须产出 `report.json` 含 `status=error` 行 |
| `pass` | 全部 Check ID severity=`fail` 通过,且无 `error` | 退出码 0 |

#### 4.4.2 HARD-FAIL 流程

`fail` 触发时:

1. **abort export** — 已生成的 codegen 产物保留在临时目录但**不**写入 `export/firmware/<module>/` 最终路径(避免下游误用)。
2. **emit `report.json`**:JSON 格式,schema per §4.5.1,含全部 Check ID 行(`status=pass` / `fail` / `warn` / `error`)。
3. **emit `report.md`**:Markdown 摘要,schema per §4.5.2,人类可读,先列 fail 行,再列 warn,再列 pass 计数。
4. **退出非零**(per §4.4.1 的退出码表)。
5. **`Affected by upstream change` 标记**:若 fail 是因为 firmware tip 漂移检出,模型仓侧 PR 须沿 [01-design-relationships.md](../01-design-relationships.md) 出边对受影响下游加 `Affected by upstream change` 标记(per RULES §10)。

#### 4.4.3 WARN 流程

`warn` 触发(无 fail)时:

1. **不 abort**:export 完成,产物落到 `export/firmware/<module>/`;PR 可合入;release 可发布。
2. **report 中 advisory 段落**:human-readable 段说明 warn 行内容。
3. **CI 上以 status-check `success-with-warnings` 标识**(per §4.6 status-check name 由 I3 锁)。
4. **保存到决策日志**:每条 warn 行记录到 `INDEX.md` 的"决策日志"段落,引用本次 run 的 `report.json` 路径(per [01-design-relationships.md §7 变更回溯规则](../01-design-relationships.md))。

#### 4.4.4 Triage path

任何 fail / warn 出现时的 review 路径:

| 情形 | Review 责任人 | 看的 artifact | 判定方向 |
|---|---|---|---|
| (a)/(c) 字段顺序 / 字节宽度 fail | B 区 author + B4 reviewer | `report.json` `coverage[].diff_text`;模型仓 `B1` / `B3` 字段表;firmware `*_types.h` | 模型仓侧改 schema(走 RULES §10 变更日志)/ firmware 侧已变更等模型仓 lockstep |
| (b) enum 数值 fail (model-only addition) | B 区 author + B2 reviewer | `report.json`;`B2 §4.2 P-1..P-3` drift policy | 拒绝模型仓单方面 enum 改动;走 firmware 主导的 lockstep 流程 |
| (b) enum warn (firmware-only end-of-list) | B 区 author + INDEX 维护方 | `report.json`;`B2 §4.2 P-2 / P-4` 末尾追加规则 | 模型仓在下个镜像周期(per [B5](B5-ins-bus-mirror.md) / [I2](../I-tooling/I2-bus-enum-mirror.md))同步 firmware 末尾追加;在此期间 warn 持续 |
| (d) `period` 数值 fail | A 区 author(架构 v1 §13)+ B3 reviewer | `report.json`;架构 v1 §13 周期表 | 周期表是架构契约;改动需架构 v1 增量更新 + INDEX 决策日志登记 |
| (e)/(f) 符号丢失 fail | I 区 author(I4 codegen 配置)+ A 区 author(A6 init 契约)| codegen 配置文件;I4 设计文档;ELF symbol dump | I4 codegen 配置错误最常见;改 I4 配置重 export |
| (a)/(c)/(d) 字节估算 warn (B4-c04) | B3 author | `report.json`;B3 §4.10.1 估算 vs firmware sizeof | 仅更新估算文档,不阻塞 |
| `error` (tool 故障) | I3 author + 触发者 | I3 错误日志 / stack trace | 修 I3 实现;重跑 |

#### 4.4.5 `--allow-drift` 例外通道

仅在以下三种情形下允许使用 `--allow-drift` 标志(由 I3 实现,per §4.6 cross-ref):

1. **Bootstrap 阶段**:模型仓首次 export 时,firmware tip 可能比模型仓 schema 旧/新;允许首次单次跨过 fail 闸门以建立 baseline。
2. **Lockstep 等待窗**:firmware 已发起契约变更但模型仓未完成镜像;在此窗内允许 export 继续以完成上下游协同测试。窗口由人工评估,不超过一个镜像周期。
3. **架构升级窗**:架构 v1 周期表 / 契约面变更已在 INDEX 决策日志登记并已发布;在此期间允许 fail 通过。

**`--allow-drift` 强制要求(审计痕迹)**:

1. **必须显式声明 reason**:`--allow-drift --reason "<text>"` 是组合参数;无 reason 时 I3 拒绝执行(per §4.6)。
2. **必须记录 INDEX 决策日志**:每次 `--allow-drift` 触发时,运行者必须在同一 PR / commit 中追加 INDEX 决策日志条目,引用 run 的 `report.json` 路径与 reason。
3. **必须填写 expiry**:`--allow-drift --reason ... --expires <YYYY-MM-DD>`;过期未解除则下次 run 会自动恢复 fail。
4. **不得用于 (e)/(f) 符号存在性**:入口符号 / PARAM 全局符号缺失是 unconditional fail;`--allow-drift` 对 e01..e03 / f01..f03 无效(I3 实现强制)。
5. **不得用于 main / release branch**:`--allow-drift` 仅在 feature branch / bootstrap 流程上有效;CI 在 main / release branch 上忽略此 flag(I3 实现强制)。

### 4.5 Report format

#### 4.5.1 JSON schema (`report.json`)

```text
{
  "tool_version":     "<semver of I3>",
  "firmware_commit":  "<sha40>",
  "model_commit":     "<sha40>",
  "run_at":           "<ISO-8601 UTC>",
  "trigger":          "pre-export | post-export | pre-merge | pre-release",
  "vehicle":          "<multicopter | fixwing | ... | * if all>",
  "overall_status":   "pass | warn | fail | error",
  "exit_code":        0 | 1 | 2 | 3,
  "allow_drift": {
    "active":   <bool>,
    "reason":   "<string|null>",
    "expires":  "<YYYY-MM-DD|null>"
  },
  "coverage": [
    {
      "id":         "B4-a01",
      "clause":     "a",
      "name":       "<short name>",
      "severity":   "fail | warn",
      "status":     "pass | fail | warn | error",
      "expected":   "<canonical text representation of B1/B2/B3 baseline>",
      "actual":     "<canonical text representation of firmware tip / codegen product>",
      "diff_text":  "<unified-diff-style or row-by-row text>",
      "source_model":    "<file:section|line ref into B1/B2/B3>",
      "source_firmware": "<firmware path:line>"
    },
    ...
  ],
  "summary": {
    "total":  24,
    "pass":   <int>,
    "warn":   <int>,
    "fail":   <int>,
    "error":  <int>
  }
}
```

**字段约束**:

- 顶层必填(无 nullable):`tool_version`, `firmware_commit`, `model_commit`, `run_at`, `trigger`, `overall_status`, `exit_code`, `coverage`, `summary`。
- `coverage[]` 长度 = §4.1 矩阵当前条目数(初版 24);未来新增 Check ID 时 `tool_version` bump,模式向前兼容(消费者按 `id` 索引)。
- `coverage[].id` 全局唯一,匹配 §4.1 表;`coverage[].clause` 取值集合 = `{"a", "b", "c", "d", "e", "f"}`。
- `coverage[].diff_text` 在 status=pass 时**可**为空字符串(consumer 解析方按空 / 非空判定);在其他 status 时**必填**且非空。
- 任何 Check ID 在 `pre-export` trigger 下若 §4.3 表标"不参与",其 `status` 应为 `"skipped"`(注:`skipped` 不计入 summary 任何计数,不触发 silent-skip 守则,因为是显式按 trigger 跳过);**当前 schema 把 `skipped` 视作第 5 种 status 值**(consumer 按需统计)。

#### 4.5.2 Markdown schema (`report.md`)

```text
# Contract Diff Report

- **Tool version:** <semver>
- **Firmware commit:** <sha40> (path: FMT-Firmware/...)
- **Model commit:** <sha40>
- **Run at:** <ISO-8601 UTC>
- **Trigger:** <pre-export | post-export | pre-merge | pre-release>
- **Vehicle:** <vehicle>
- **Overall status:** **<PASS | WARN | FAIL | ERROR>**
- **Exit code:** <0..3>
- **Allow-drift:** <inactive | active(reason="<...>", expires=<...>)>

## Summary

| Total | Pass | Warn | Fail | Error |
|---:|---:|---:|---:|---:|
| 24 | <n> | <n> | <n> | <n> |

## Failures (severity=fail, status=fail)

| Check ID | Clause | Name | Expected | Actual | Diff (excerpt) |
|---|---|---|---|---|---|
| <id> | <a..f> | <short> | <text> | <text> | <text> |

## Warnings (severity=warn or warn-eligible)

| Check ID | Clause | Name | Expected | Actual | Note |
|---|---|---|---|---|---|

## Errors (tool-side)

| Check ID | Reason |
|---|---|

## Pass list (collapsed)

<count> checks passed: <id1>, <id2>, ...
```

**Markdown schema 规则**:

1. Failures 段在最前(易于扫读 fail 原因)。
2. Pass list 折叠为单行 ID 列表,避免淹没 fail 信息。
3. 所有 ID 与 §4.1 矩阵的 ID 一一对应,可点击 / 检索。
4. Markdown report 与 JSON report 同一次 run 内容一致(I3 在生成时由同一份内部状态对象渲染,per §4.6 实现指针)。

#### 4.5.3 Worked example fragment

下例展示一次 post-export run 的 JSON 片段,刻意覆盖 fail / warn / pass / error / skipped 五种 status,以验证 schema 完整性。**这只是 schema 演示;真实数据由 I3 生成**(完整 worked example 见 [I3 §4.7](../I-tooling/I3-contract-diff.md))。

```text
{
  "tool_version": "0.1.0",
  "firmware_commit": "abcdef1234567890abcdef1234567890abcdef12",
  "model_commit":    "1234567890abcdef1234567890abcdef12345678",
  "run_at": "2026-05-08T03:00:00Z",
  "trigger": "post-export",
  "vehicle": "multicopter",
  "overall_status": "fail",
  "exit_code": 2,
  "allow_drift": {"active": false, "reason": null, "expires": null},
  "coverage": [
    {
      "id": "B4-a01",
      "clause": "a",
      "name": "FMS_Out_Bus field order",
      "severity": "fail",
      "status": "fail",
      "expected": "[..., 'cmd_mask', 'status', 'state', 'ext_state', ...] (B1 §4.7.1)",
      "actual":   "[..., 'cmd_mask', 'status', 'state', 'ext_state', 'new_field_x', ...]",
      "diff_text": "+ new_field_x (firmware-side addition; not declared in B1 §4.7)",
      "source_model": "docs/design/B-contracts/B1-bus-inventory.md#§4.7",
      "source_firmware": "FMT-Firmware/src/model/fms/multicopter/lib/FMS_types.h:142"
    },
    {
      "id": "B4-b03",
      "clause": "b",
      "name": "VehicleMode end-of-list addition",
      "severity": "warn",
      "status": "warn",
      "expected": "VehicleMode members 0..7 (B2 §4.4.5)",
      "actual":   "VehicleMode members 0..8 (firmware-only addition at end)",
      "diff_text": "+ VMODE_NEW_MODE = 8  (firmware-only, end-of-list, additive per B2 §4.2 P-2)",
      "source_model": "docs/design/B-contracts/B2-enum-inventory.md#§4.4.5",
      "source_firmware": "FMT-Firmware/src/model/fms/multicopter/lib/FMS_types.h:201"
    },
    {
      "id": "B4-e01",
      "clause": "e",
      "name": "Plant entry symbols",
      "severity": "fail",
      "status": "pass",
      "expected": "{'Plant_init', 'Plant_step'} ⊆ ELF symbols",
      "actual":   "{'Plant_init', 'Plant_step', 'Plant_initialize'} ⊆ ELF symbols",
      "diff_text": "",
      "source_model": "(codegen artifact)",
      "source_firmware": "(N/A — checked on model-side artifact)"
    }
  ],
  "summary": {"total": 24, "pass": 22, "warn": 1, "fail": 1, "error": 0}
}
```

完整 worked example(覆盖 (a) `FMS_Out_Bus` 字段添加 + (b) `VehicleMode.HOLD` 数值漂移)由 I3 §4.7 给出(co-seal 兄弟兑现)。

### 4.6 Cross-reference map to I3

每条 Check ID → I3 实现章节 → 数据源映射:

| Check ID | I3 实现章节 | 数据源(模型仓侧)| 数据源(firmware 端)|
|---|---|---|---|
| B4-a01 | I3 §4.4.a (bus 字段顺序 diff) | B1 §4.2..§4.8 + I2 YAML 镜像(可选) | `FMT-Firmware/src/model/{plant,fms,control}/<vehicle>/lib/{Plant,FMS,Controller}_types.h` |
| B4-a02 | I3 §4.4.a (类型映射 diff) | B1 §4.1.2 类型表 + 各 bus Type 列 | 同上 |
| B4-a03 | I3 §4.4.a (累计字节宽度 diff) | B1 §4.2..§4.8 Width / 总宽度行 | 同上(静态计算 sizeof)|
| B4-a04 | I3 §4.4.a (endianness 检查) | B1 §4.1.1 | firmware target triple / ELF header |
| B4-a05 | I3 §4.4.a (子 schema diff) | B1 §4.6 | 同 firmware `*_types.h` 子 typedef |
| B4-b01 | I3 §4.4.b (enum 类型存在性) | B2 §4.4.x 类型清单 | firmware `*_types.h` `typedef enum` |
| B4-b02 | I3 §4.4.b (enum 数值精确比) | B2 §4.4.x 值表 | 同上 |
| B4-b03 | I3 §4.4.b (双向覆盖,含末尾追加判定) | B2 §4.4.x + B2 §4.2 drift policy | 同上 |
| B4-b04 | I3 §4.4.b (底层整型宽度) | B2 §4.4.x header underlying type | 同上 + `sizeof(<enum>)` |
| B4-c01 | I3 §4.4.c (PARAM 字段顺序) | B3 §4.3.2 / §4.4.2 / §4.5.2 `#` 列 | `FMT-Firmware/src/model/{plant,fms,control}/<vehicle>/lib/*.h` 含 `*_PARAM_TYPE` 声明 |
| B4-c02 | I3 §4.4.c (PARAM 字段类型) | B3 §4.3.2 / §4.4.2 / §4.5.2 Type 列 | 同上 |
| B4-c03 | I3 §4.4.c (PARAM storage class) | B3 §4.6.4 budget;I4 codegen 配置 | codegen 产物 `.c` / `.h` |
| B4-c04 | I3 §4.4.c (PARAM 总字节宽度) | B3 §4.10.1 估算 | firmware `sizeof(*_PARAM_TYPE)` |
| B4-d01 | I3 §4.4.d (EXPORT 字段顺序) | B3 §4.7.2..§4.7.4 | 同 c01 firmware 路径 |
| B4-d02 | I3 §4.4.d (`period` 数值) | B3 §4.7.2..§4.7.4 默认值列 + 架构 v1 §13 周期表 | firmware EXPORT 实例化 `.c` |
| B4-d03 | I3 §4.4.d (`model_info[]` 长度) | B3 §4.7.5 | firmware EXPORT typedef |
| B4-d04 | I3 §4.4.d (EXPORT 总字节宽度) | B3 §4.7 | firmware `sizeof(*_EXPORT_TYPE)` |
| B4-e01..e03 | I3 §4.4.e (入口符号扫描) | (N/A;检查目标 = 模型仓 codegen 产物) | codegen `.c` 函数定义 / ELF symbol table |
| B4-f01..f03 | I3 §4.4.f (PARAM 全局符号扫描) | (N/A;检查目标 = 模型仓 codegen 产物) | 同 e01..e03 |

I3 必须按此映射逐条实现;任一缺失 → I3 退出条件不满足。

I3 的输入 / 输出接口要点(由 B4 策略层声明,由 I3 实现细节锁):

1. **输入**:firmware 仓路径 + commit SHA(强制,避免无 SHA 锁定漂移);模型仓 B1 / B2 / B3(由文件路径或 I2 生成的 YAML 镜像);codegen 产物路径(post-export / pre-merge / pre-release 阶段必填);allow-drift flag(per §4.4.5)。
2. **输出**:`report.json`(强制,per §4.5.1)+ `report.md`(强制,per §4.5.2);写入 `<output-dir>/contract-diff/<run-timestamp>/`。
3. **退出码**:0 / 1 / 2 / 3 per §4.4.1。
4. **CI 集成 status-check name**:`contract-diff/post-export`(由 I3 锁定;per §4.6 cross-ref)。

### 4.7 Lockstep policy(per 架构 v1 §15)

承接架构 v1 §15(MIL → SIL → SIH → firmware integration)+ §17 Risk 1 / Risk 6,本节定义模型仓与 firmware tip 之间的 lockstep 同步规则。

**L-1 模型仓 PR 触发**:任何模型仓 PR 触及 [B1](B1-bus-inventory.md) / [B2](B2-enum-inventory.md) / [B3](B3-parameter-schema.md) 任一文件 → CI 必须跑 pre-merge gate(全 24 项 Check ID,trigger=`pre-merge`)。Check ID 失败阻塞 PR。

**L-2 firmware tip 漂移检测**:模型仓 PR 跑 pre-merge gate 时,即使 PR 本身未触及 B1/B2/B3,**只要** firmware tip 自上次 pre-merge gate 跑过以来发生变化,**且**新跑产生 fail/warn 行,触发以下流程:

1. 在 `INDEX.md` 决策日志追加一条记录:`<日期> firmware tip 漂移检出 (run=<run id>)`,引用 `report.json` 路径。
2. 沿 [01-design-relationships.md §4](../01-design-relationships.md) 受影响下游加 `Affected by upstream change` 标记(per RULES §10)。
3. 若漂移属 fail 级:阻塞当前 PR,要求模型仓 schema 同步后再合入;若漂移属 warn 级(B4-b03 firmware-only end-of-list):允许 PR 合入,但同步任务在下个镜像周期完成(per [B5](B5-ins-bus-mirror.md) / [I2](../I-tooling/I2-bus-enum-mirror.md))。

**L-3 镜像周期**:模型仓与 firmware tip 之间的镜像周期由 [B5](B5-ins-bus-mirror.md) / [I2](../I-tooling/I2-bus-enum-mirror.md) 定义;B4 不规定具体节奏,但要求每个镜像周期结束时 pre-merge gate 必须 pass(否则镜像未完成)。

**L-4 Release 锁定**:每个模型仓 release tag 必须挂一份与 firmware release SHA 对应的 pre-release gate report.json(in `export/reports/`)。Release manifest 引用 `firmware_commit` 字段。

**L-5 Bootstrap 例外**:首次模型仓 export 允许使用 `--allow-drift` 跨过 fail(per §4.4.5);但必须在 INDEX 决策日志登记 baseline run。Bootstrap 完成后 lockstep 即生效。

**L-6 单仓变更登记**:任何只在模型仓侧改动 B1/B2/B3 字段 / enum 数值的 PR(即使 firmware 未变),也必须按 RULES §10 在 INDEX 决策日志登记。

### 4.8 设计决策汇总

| 决策 | 内容 | 依据 |
|---|---|---|
| **D-1 矩阵 6 clauses** | 覆盖矩阵以 (a)..(f) 6 大类组织,涵盖 bus / enum / PARAM / EXPORT / 入口符号 / PARAM 全局符号 | 任务 brief audit-fortified 五条 + (f) 任务追加;闭合 [00-design-plan §4.B B4 退出条件](../00-design-plan.md) (a)..(e)|
| **D-2 24 项 Check ID** | 矩阵展开为 24 行,每行有唯一 ID,与 I3 §4.4 子节一一对应 | §4.6 cross-ref 表;细粒度便于报告与 triage |
| **D-3 零 tolerance** | 全部 Check ID 在数值 / 字节级 tolerance = 0;不允许"近似相等" | 架构 v1 §5.1.1 contract-first;B2 §4.2 P-1 不可变值 |
| **D-4 22 fail / 2 warn** | 默认 severity=fail;仅 (b) firmware-additive-end-of-list 与 (c) 字节估算 = warn | B2 §4.2 P-2 末尾追加规则;B3 §4.10.1 估算说明 |
| **D-5 4 触发点 × 24 矩阵** | pre-export(轻量子集)/ post-export / pre-merge / pre-release(全量)| 架构 v1 §14.4 contract review gates;§15 lockstep |
| **D-6 HARD-FAIL 默认** | 任一 fail 阻塞 export;`--allow-drift` 仅限 bootstrap / lockstep 等待 / 架构升级三种情形,需 reason+expiry+INDEX 登记 | 架构 v1 §17 Risk 1;不允许 silent override |
| **D-7 报告强制双格式** | JSON + Markdown,同一次 run 同源;JSON 给 CI / 自动化,Markdown 给人类 review | 任务 brief §4.5;消费者多元(CI / GCS team / firmware team) |
| **D-8 status-check name** | `contract-diff/post-export`(主)+ `contract-diff/pre-merge` / `contract-diff/pre-release`(分支别名)| 由 [I3 §4.6](../I-tooling/I3-contract-diff.md) 实现锁;B4 在此声明命名规则 |
| **D-9 与 I3 协同** | B4 给 WHAT,I3 给 HOW;§4.6 cross-ref 表确保每条检查可追踪到 I3 实现 | co-seal batch (B4, I3) per [01-design-relationships.md §5](../01-design-relationships.md) |
| **D-10 Lockstep 规则 L-1..L-6** | 把架构 v1 §15 + §17 Risk 1/6 落到具体 PR / release / bootstrap 流程 | §4.7 + 任务 brief lockstep 要求 |

## 5. 已知风险与悬而未决问题

- **firmware commit hash 暂缺**
  - 影响:[RULES §5 / §3 表](../RULES.md) require 镜像自 firmware 的契约记录 commit hash + 文件相对路径;本文件 §3 给路径占位 `FMT-Firmware @ <pending hash>`(per B1/B2/B3 同样的占位约定)。
  - 处置:open;评审通过前由 reviewer / orchestrator 在 INDEX 决策日志登记本工作项时补齐。本工作项 contract_impact=yes 已声明。

- **B4-a03 / d04 累计字节宽度 vs natural alignment**
  - 影响:B1 §4.1.3 声明 natural alignment;但 firmware 编译器若使用 `__attribute__((packed))` 或显式 `_pad[N]`,B4-a03 / d04 报 fail。
  - 处置:首跑 pre-merge gate 时若发现 alignment 差异,作者侧选项二选一:(1) 模型仓 schema 加显式 pad 字段(走 B1 / B3 变更日志);(2) firmware 改 alignment(契约破坏,走 INDEX 决策日志 + 架构 v1 §17 Risk 1 跟踪)。

- **B4-b03 双向覆盖判定的边界条件**
  - 影响:`firmware-only addition at end` 的"end-of-list"判定依赖 enum 数值连续递增。若 firmware 跳号(例如保留区间 0x100..0x1FF)或非递增(例如填补 deprecate 留下的洞),B4-b03 算法判定有误判风险。
  - 处置:I3 §4.4.b 实现要为 jump / deprecate hole 写明判定规则;在不确定时降级到 `fail`(safe default)。本风险在首跑 pre-merge gate 时自然暴露。

- **`--allow-drift` 滥用风险**
  - 影响:即使有 reason + expiry + INDEX 登记,反复滥用会让 fail 闸门失效。
  - 处置:CI 上加聚合监控(`--allow-drift` 触发次数 / 月);超阈值在 INDEX 月度报告中标记。本设计层面只声明规则;监控实现由 I5(sim runner / CI gate 配套)负责。

- **codegen 产物 ELF 可用性依赖**
  - 影响:B4-e01..e03 / f01..f03 的 acceptance criterion 优先用 ELF symbol table;若 codegen 流程没产生 ELF(例如在 nightly fast loop 上仅生成 `.c`),退化到 `.c` 静态扫描。
  - 处置:I3 必须实现两路检查(ELF + `.c` fallback);若两路都无法读到检查目标,出 `error` 不出 `pass`(per §4.5.1 silent-skip 守则)。

- **firmware tip 在 CI 上如何引用**
  - 影响:pre-merge gate 引用 firmware tip(默认分支 HEAD),但 firmware 仓未挂载在本设计阶段环境。CI 实施时 firmware 路径 / SHA 来源由 I3 + I5 解决。
  - 处置:本文件 §4.6 声明输入接口需含 `--firmware-repo-path` + `--firmware-sha`;具体如何 fetch firmware 仓由 I3 入参 / CI 配置决定。

- **多 vehicle 场景下的 firmware 路径**
  - 影响:firmware `*_types.h` 路径含 `<vehicle>` 段(`multicopter` / `fixwing` / ...);Phase 2 仅 multicopter,但未来扩展时同一份 B1 / B2 / B3 vs 多组 firmware 路径如何 diff 未规定。
  - 处置:本设计层面只声明 Phase 2 multicopter;多 vehicle diff 矩阵留 [架构 v1 §17 OQ1](../../architecture/2026-05-05-fmt-model-architecture-v1.md) / A5 后续修订;I3 入参 `--vehicle` 已预留扩展空间。

- **B4-c03 storage class 检查的 firmware 端不对称**
  - 影响:storage class 是模型仓 codegen 产物概念;firmware 不一定显式标(C 源代码无 `extern volatile` 在 enum 类型上的明确语义)。
  - 处置:B4-c03 acceptance 仅验证模型仓侧 codegen 产物 vs B3 §4.6.4 budget 一致;不与 firmware 对称比对。本风险在 §4.1 / §4.2 已显式说明。

- **本工作项 contract_impact=yes 的影响范围**
  - 影响:任何 B4 策略变更 = firmware 契约保护范围变更;须经 INDEX 决策日志登记(per RULES §10)。
  - 处置:已声明 contract_impact=yes;变更走 RULES §10 + INDEX 登记。本文件不直接定义契约字段(由 B1 / B2 / B3 拥有),仅定义 diff 守则,但守则放宽 / 收紧直接影响契约保护强度。

- **co-seal batch 协同风险**
  - 影响:I3 兄弟 draft 在协同评审中可能提出对 §4.5 JSON schema / §4.6 cross-ref 的反馈;本文件需随之更新。
  - 处置:per RULES §6 self-check 第 2 项 + 01-design-relationships.md §5 entry 3,co-seal batch (B4, I3) 在 Wave 5 协同定稿;评审时 reviewer 检查策略 / 实现两侧字段对齐。

## 6. 退出条件复核

B4 在 [`00-design-plan.md §4.B`](../00-design-plan.md) 中的退出条件原文:

> diff 工具(由 I3 实现)的输入/输出格式、运行时机、失败响应规则;**必须覆盖:(a) 所有 bus 字段顺序与类型字节相等;(b) 所有 enum 数值精确相等;(c) `FMS_PARAM`/`CONTROL_PARAM`/`PLANT_PARAM` 字段顺序与类型相等;(d) `FMS_EXPORT`/`CONTROL_EXPORT`/`PLANT_EXPORT` 字段顺序、`period` 数值、`model_info[]` 长度相等;(e) 符号存在性检查 — `FMS_init`/`FMS_step`/`Controller_init`/`Controller_step`/`Plant_init`/`Plant_step` 必须出现在生成产物中**;明确 pass/fail 判定阈值

逐条复核(audit-fortified 五子句 (a)..(e) + 任务追加 (f) + pass/fail 阈值):

| # | 退出条件分解 | 本文档依据 | 状态 |
|---|---|---|---|
| 1 | 输入 / 输出格式 | §4.5.1 JSON schema + §4.5.2 Markdown schema + §4.5.3 worked example fragment;§4.6 输入 / 输出接口要点 | 满足 |
| 2 | 运行时机 | §4.3 4 触发点表 + 触发点关系图 + Run timing 规则 1..5 | 满足 |
| 3 | 失败响应规则 | §4.4.1 失败级别 → 后果表;§4.4.2 HARD-FAIL 流程;§4.4.3 WARN 流程;§4.4.4 Triage path 表;§4.4.5 `--allow-drift` 规则 | 满足 |
| 4 | (a) 所有 bus 字段顺序与类型字节相等 | §4.1 矩阵 B4-a01..a05(bus 字段顺序 / 类型映射 / 累计字节宽度 / endianness / 子 schema)5 行 covering B1 §4.2..§4.8 16 / 16 bus | 满足 |
| 5 | (b) 所有 enum 数值精确相等 | §4.1 矩阵 B4-b01..b04(enum 类型存在性 / 数值精确 / 双向覆盖 / 底层整型宽度)4 行 covering B2 §4.4.1..§4.4.16 全 enum | 满足 |
| 6 | (c) `*_PARAM` 字段顺序与类型相等 | §4.1 矩阵 B4-c01..c04(PARAM 字段顺序 / 类型 / storage class / 总字节宽度)4 行 covering B3 §4.3 / §4.4 / §4.5 三个 PARAM struct | 满足 |
| 7 | (d) `*_EXPORT` 字段顺序、`period` 数值、`model_info[]` 长度 | §4.1 矩阵 B4-d01..d04(EXPORT 字段顺序 / `period` 数值 / `model_info[]` 长度 / 总字节宽度)4 行 covering B3 §4.7.2..§4.7.5 三个 EXPORT struct | 满足 |
| 8 | (e) 入口符号存在性 | §4.1 矩阵 B4-e01..e03(Plant / FMS / Controller `*_init` + `*_step` 必现)3 行;§4.2 acceptance "ELF symbol table 含 X AND Y" | 满足 |
| 9 | (f) PARAM 全局符号存在性(任务 brief audit-fortified 追加)| §4.1 矩阵 B4-f01..f03(`PLANT_PARAM` / `FMS_PARAM` / `CONTROL_PARAM` 全局名必现)3 行;§4.2 acceptance | 满足 |
| 10 | 明确 pass/fail 判定阈值 | §4.2 阈值表(每行给 severity + tolerance + acceptance criterion);§4.4.1 退出码表(pass=0 / warn=1 / fail=2 / error=3)| 满足 |

附:对任务 brief 显式额外要求的复核:

| 任务 brief 要求 | 本文档依据 | 状态 |
|---|---|---|
| §4.1 单一 canonical coverage 表 | §4.1 表(24 行)+ 矩阵覆盖证明子表 | 满足 |
| §4.2 severity / tolerance / acceptance | §4.2 24 行表 | 满足 |
| §4.3 pre-export / post-export / pre-merge / pre-release 四级触发 | §4.3 4 触发点表 + 关系图 | 满足 |
| §4.4 失败响应(HARD-FAIL / WARN / 三角化 / `--allow-drift` 规则)| §4.4.1..§4.4.5 5 子节 | 满足 |
| §4.5 JSON schema + Markdown 表 + worked example fragment | §4.5.1 / §4.5.2 / §4.5.3 三子节 | 满足 |
| §4.6 与 I3 cross-ref 表 | §4.6 24 行表 + I3 输入 / 输出接口要点 | 满足 |
| §4.7 lockstep policy + INDEX 决策日志触发 | §4.7 L-1..L-6 规则 | 满足 |

附:对架构 v1 / A1 / B1 / B2 / B3 缺口闭合的复核:

| 来源 | 缺口 | 本文档闭合位置 |
|---|---|---|
| 架构 v1 §17 Risk 1 (contract drift) | 字段顺序 silent drift 未给检测机制 | §4.1 (a) 覆盖 + §4.7 lockstep |
| 架构 v1 §17 Risk 6 (parameter governance) | tuning divergence 检测未给机制 | §4.1 (c) + B3 §4.8.1 已转交 B4 |
| 架构 v1 §10 silent enum drift | enum 数值漂移未给检测机制 | §4.1 (b) 覆盖 + §4.2 B4-b03 双向判定 |
| A1 §5.1.7 / §5.1.9 critical | 字段顺序 / enum drift 检测 | §4.1 (a) / (b) 覆盖 |
| B1 §4.10 与 B4 退出条件呼应表 | B1 给 B4 的输入 baseline | §4.1 矩阵 B4-a01..a05 / b01 直接消费 |
| B2 §4.2 P-1..P-6 drift policy | 模型仓侧 drift 政策的运行时兑现 | §4.2 B4-b03 / B4-b04 acceptance |
| B3 §4.8 diff coverage 要求 | PARAM / EXPORT diff 必跑项 | §4.1 矩阵 B4-c01..c04 / d01..d04 |

退出条件全部满足;状态:`draft`,待评审升级到 `reviewed`。

## 7. 下游影响

按 [`01-design-relationships.md §4.2 / §4.9 / §5`](../01-design-relationships.md) 本工作项的出边:

```text
B1, B2, B3 → B4
B4 ↔ I3                              (co-seal batch entry 3)
(间接)B4 ⇢ I4 / I5 / 整个 export 链
```

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [I3 契约 diff 脚本设计](../I-tooling/I3-contract-diff.md) *(co-seal sibling, draft)* | ↔ | §4.1 24 行覆盖矩阵 + §4.5 报告 schema + §4.6 cross-ref 映射 → I3 §4.4 实现规约;§4.2 阈值表 → I3 退出码 / status-check 配置;§4.4.5 `--allow-drift` 规则 → I3 CLI 设计;§4.7 lockstep → I3 CI 配置 |
| [I4 Codegen 配置脚本设计](../I-tooling/I4-codegen-config.md) *(未启动,Wave 12)* | ⇢ | §4.1 (e) 入口符号 + (f) PARAM 全局符号要求 → I4 codegen 配置必须保证生成的 entry 符号与 PARAM 全局符号可被 ELF 扫描到;§4.2 B4-c03 storage class 一致性 → I4 storage class 配置必须与 B3 §4.6.4 budget 匹配 |
| [I5 仿真运行/批量回归脚本设计](../I-tooling/I5-sim-runner.md) *(未启动,Wave 12)* | ⇢ | §4.3 触发点 → I5 在 export 流程中的何处插入 contract diff stage;§4.4.1 退出码 → I5 错误处理 |
| [B5 INS_Out_Bus 镜像同步流程](B5-ins-bus-mirror.md) *(未启动,Wave 5,与 B4 并行)* | (无直接出边)| (B4 把 `INS_Out_Bus` 视为 B1 §4.3.1 一条 bus,统一按 (a) 规则 diff;B5 owns 镜像源单向 / 共享子模块决策,与 B4 解耦)|
| [I2 Bus/enum 镜像脚本设计](../I-tooling/I2-bus-enum-mirror.md) *(未启动,Wave 5)* | ⇢ (可选输入) | §4.6 数据源列声明 I3 可选地以 I2 生成的 YAML 镜像作为 baseline 输入 → 若 I2 实现 YAML mirror,B4 / I3 直接消费;若 I2 仅生成 markdown,B4 / I3 自行解析 B1 / B2 / B3 markdown 表 |
| **架构 v1 §17 Risk 1 / 6**(架构层风险跟踪)| ⇢ | §4.7 lockstep + §4.1 (a) (b) (c) 覆盖项闭合两条 critical 风险的检测面 |
| **INDEX.md 决策日志**(契约风险登记入口)| ⇢ | §4.4.5 `--allow-drift` 触发 / §4.7 L-2 firmware tip 漂移检出 / L-6 单仓变更登记三个 trigger 都驱动 INDEX 决策日志条目 |
| 全部 C / D / E / F / G 区下游(间接)| (变更回溯)| 任何 B4 fail / firmware tip 漂移触发 RULES §10 受影响下游 `Affected by upstream change` 标记;沿 [01-design-relationships.md](../01-design-relationships.md) 出边自动传播 |

整体而言:**B4 本身无强前置下游(I3 是协同兄弟);但 B4 是契约保护链上的 final guardrail,影响所有 export-触发的 contract change 流程**。

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-07 | B4 author | 初稿(co-seal batch (B4, I3) Wave 5)|

## Self-check

- [x] frontmatter 完整,字段值合法
- [x] 上游文档全部存在且 status ≥ reviewed,**或**为本工作项所在 co-seal batch 的同批兄弟(架构 v1 已发布;A1 / A2 / A3 / A6 reviewed 2026-05-05;B1 / B2 / B3 reviewed 2026-05-07;I3 为 co-seal batch (B4, I3) 同批兄弟,allowed per RULES §6 第 2 项;§3 已注明 batch 名)
- [x] 退出条件逐条复核完成,每条均给出依据(§6 主退出条件表 10 项 + 任务 brief 显式额外要求 7 项 + 缺口闭合 7 项)
- [x] 引用路径全部可点击访问
- [x] 不存在 [RULES §5](../RULES.md) 禁则中的内容(无 .slx 截图;无可执行 .m / Python — §4.5 worked example 块为 schema 演示 JSON 文本不是可执行代码;无 firmware 实现复述,只描述模型仓侧契约保护策略;不重定义 bus/enum/PARAM 字段表 — 委托 B1/B2/B3;FMT-Firmware 引用以路径占位 + `<pending hash>` per B1/B2/B3 模式)
- [ ] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记 — **pending orchestrator INDEX 登记**(本文件作者按 fmt-design-author skill 规则不直接编辑 INDEX;orchestrator 在 review pass 后追加条目并补 firmware commit hash;同 [B1 / B2 / B3 同期模式](B1-bus-inventory.md))
- [ ] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径 — **pending**:文件相对路径已在 §3 给出(`FMT-Firmware/src/model/{plant,fms,control}/<vehicle>/lib/{Plant,FMS,Controller}_types.h` + `FMT-Firmware/src/model/ins/lib/INS_types.h`);commit hash 占位 `<pending hash>` 与 [B1 §3 / B2 §3 / B3 §3](B1-bus-inventory.md) 同期补齐。本文件不创建镜像产物(由 [B5](B5-ins-bus-mirror.md) / [I2](../I-tooling/I2-bus-enum-mirror.md) 落地)
- [x] 下游影响已沿关系图识别完毕(§7 含 I3 协同 + I4 / I5 / I2 / B5 间接 + 架构 v1 / INDEX / 全部 C/D/E/F/G 区受变更回溯影响)
- [x] 文档不超出本工作项范围(无越权设计:bus 字段表留 B1;enum 数值留 B2;PARAM/EXPORT 字段表留 B3;脚本 CLI / 实现 / parser 留 I3;codegen 配置留 I4;sim runner CI 集成留 I5;mirror 流程留 B5 / I2;cmd_mask 位号留 D6/E1)
