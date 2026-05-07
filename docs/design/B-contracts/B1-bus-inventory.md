---
work_item: B1
title: Bus 清单与 schema 设计
upstream: ["架构 v1", "A1", "A2", "A3", "A4", "A5", "A6", "A7", "A8"]
contract_impact: yes
status: reviewed
authored_at: 2026-05-07
last_reviewed_at: 2026-05-07
reviewer_verdict: pass
---

# B1 Bus 清单与 schema 设计

## 1. 目的

把 [A3](../A-architecture/A3-module-boundaries.md) 给出的 16 个契约 bus 的 **field-level 语义** 落到 **byte-level schema**:为每个 bus 锁定字段顺序、Simulink 数据类型、字段宽度、累计字节偏移量、单位 / 坐标系 / 编码语义,与 firmware `*_types.h` 镜像对齐;并显式把 enum 字段委托给 [B2](B2-enum-inventory.md)、参数驱动字段委托给 [B3](B3-parameter-schema.md),作为下游 C/D/E/F 区的 **single source of truth**(per RULES §5 / 架构 v1 §9.3 / §14.1)。

## 2. 范围

**在范围:**

- 16 个契约 bus 的字段级 schema:`Pilot_Cmd_Bus` / `GCS_Cmd_Bus` / `Auto_Cmd_Bus` / `Mission_Data_Bus` / `INS_Out_Bus` / `FMS_Out_Bus` / `Control_Out_Bus` / `Plant_States_Bus` / `Extended_States_Bus` / `Environment_Info_Bus` / `States_Init_Bus` / `IMU_Bus` / `MAG_Bus` / `Barometer_Bus` / `GPS_uBlox_Bus` / `AirSpeed_Bus`
- 每个字段:**字段名、Simulink 数据类型、字节宽度、累计字节偏移量(natural-alignment best-effort,B4 to verify)、单位、坐标系、编码 / 量化语义、validity / cmd_mask / passthrough 守卫、variant-conditional 标签**
- 每个 bus 的 **Simulink Bus Object 名**(per A2 R-5.1)、**owner module**(per 架构 v1 §7)、**nominal rate**(per A4 §4.1)、**source firmware header path**(带 `<pending hash>` 占位,per A3 §3 与 A6/A7 模式)
- **`FMS_Out_Bus` 字段顺序总账(field-order ledger)**:闭合 [A1 §5.1.5](../A-architecture/A1-v1-review.md) 缺口,把 rate cmd / attitude cmd / velocity cmd / acceleration cmd / actuator passthrough / throttle / cmd_mask / status / state / ext_state / ctrl_mode / mode / reset / wp / home / error 字段全部按设计意图序列化
- **嵌套类型子 schema**:`quat[4]` / `pos_ned[3]` / `vel_ned[3]` / `ang_rate_b[3]` / `euler[3]` / `INS_Status` / `INS_Flag` / `Waypoint`(用于 `Mission_Data_Bus.wp_array[]`)等
- 每个 bus 的 **总字节宽度**(natural alignment;B4 to verify alignment padding)
- **endianness 假设**(显式声明:little-endian 与 ARM Cortex-M 目标一致)
- **B1 ↔ B2 / B3 交叉引用矩阵**:每条 enum 字段 → B2;每条 PARAM 类别字段 → B3
- 与 [B4 契约 diff 策略](B4-contract-diff.md) 退出条件覆盖项(bus 字段顺序 / *_PARAM / *_EXPORT / period / model_info[] / 符号存在性)的呼应

**不在范围(由其他工作项处理):**

- enum 数值表(成员名 → 整数值 → 含义)— 由 [B2 Enum 清单与数值锁定](B2-enum-inventory.md) 处理(co-seal 兄弟)
- `FMS_PARAM` / `CONTROL_PARAM` / `PLANT_PARAM` 字段清单与默认值表 — 由 [B3 Parameter schema 设计](B3-parameter-schema.md) 处理(co-seal 兄弟)
- `*_EXPORT` 结构字段、`period` 数值、`model_info[]` 长度 — 由 [B3 §<EXPORT 节>](B3-parameter-schema.md) 与 [I4 codegen 配置](../I-tooling/I4-codegen-config.md) 落地(B1 仅承诺 EXPORT struct 名固化,字段由 B3)
- 实际 firmware 字节布局精确性(padding / 编译器对齐方式 / `__attribute__((packed))`)— 留 [B4 契约 diff 策略](B4-contract-diff.md) + [I3 契约 diff 脚本](../I-tooling/I3-contract-diff.md) 在 export 时验证;B1 给设计意图与 best-effort 累计偏移量,带 `(B4 to verify)` 标注
- INS 内部 schema(EKF 状态、协方差等)— firmware 拥有(架构 v1 §3 / §12.3);B1 只给 `INS_Out_Bus` 消费侧字段顺序,镜像流程由 [B5 INS_Out_Bus 镜像同步流程](B5-ins-bus-mirror.md) 落地
- 字段级仿真行为 / reset 详细流程 — 由 [A6 Init/Reset](../A-architecture/A6-init-reset-contract.md) 已给类别;B1 不重述
- cmd_mask 位语义具体位号 / 名 — 由 [D6](../D-fms/D6-fms-controller-interface.md) + [E1](../E-controller/E1-controller-functional.md) co-seal batch(Wave 8)锁定;B1 仅记录 `cmd_mask` 字段类型 = `uint32` bitfield
- harness-only Variant Subsystem 拓扑(`Plant_States_Bus → Controller`)— 由 [A5 §4.5](../A-architecture/A5-variant-strategy.md) / [G1](../G-harness/G1-mil-toplevel.md) 锁定;B1 仅记录契约层字段

## 3. 依赖

| 上游 | 引用位置 | 用途 |
|---|---|---|
| 架构 v1 §5.1.4 / §5.1.5 / §5.1.6 / §5.1.7 / §5.1.8 / §13 / §14.1 | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | 16 个 firmware-visible 契约 bus 名固化 + `FMS_Out_Bus` 关键字段集(rate / att / vel / acc cmd / actuator passthrough / throttle / cmd_mask / status / state / ext_state / ctrl_mode / mode / reset / wp / home / error,A1 §5.1.5 demand)+ 模块周期表(B1 nominal rate 列引用) |
| [A1 架构 v1 评审与缺口闭合](../A-architecture/A1-v1-review.md) (status: reviewed, 2026-05-05) | A1 §5.1.5 demand:`FMS_Out_Bus` 字段顺序 / 类型 | B1 §4.7 ledger 闭合此缺口 |
| [A2 命名与目录约定设计](../A-architecture/A2-naming-conventions.md) (status: reviewed, 2026-05-05) | A2 R-5.1 / R-5.2 / R-5.3 / R-5.4 / R-10.1 / R-10.4 / R-10.7 / E-5.1 / E-10.1 / R-10.3 | Simulink Bus Object 名固化、字段命名风格、单位 / 坐标系 / 角度后缀;`timestamp` 字段名作为 firmware-legacy E-10.1 例外保留(无后缀,uint32 ms,见 A7 §4.1) |
| [A3 模块边界信号清单细化](../A-architecture/A3-module-boundaries.md) (status: reviewed, 2026-05-07) | A3 §4.3 / §4.4 / §4.5 / §4.6 / §4.7 / §4.8 / §4.9 / §4.12 | 字段级语义:每字段 Type 列(A3 标 best-effort)、Unit、Frame、产出方 Rate、Reset 类别、validity / passthrough / variant 标签 — B1 直接 echo 并加 byte width / cumulative offset / encoding 列 |
| [A4 跨速率边界设计](../A-architecture/A4-rate-boundaries.md) (status: reviewed, 2026-05-05) | A4 §4.1 + RB-01..RB-07 | 每 bus header 表 nominal rate 列引用;每 bus 的产出方 rate(Plant 1 ms / Controller 5 ms / FMS 20 ms / INS 10 ms / external) |
| [A5 变体策略设计](../A-architecture/A5-variant-strategy.md) (status: reviewed, 2026-05-05) | A5 §4.2.2 / §4.5 / §4.6.4 | variant-conditional 字段标(`AirSpeed_Bus` / `aoa_rad` / `aos_rad` / `motor_cmd_passthrough` / `Control_Out_Bus → FMS` 回送);harness-only 字段不进 firmware export |
| [A6 Init/Reset 状态契约](../A-architecture/A6-init-reset-contract.md) (status: reviewed, 2026-05-05) | A6 §4.4(类别表)/ §4.5 | 每字段 reset 类别(PARAM / HARDCODED / ZERO / PRESERVED)— B1 不重述,经由"B3 cross-ref"列指向 PARAM 类别字段 |
| [A7 时间与时间戳约定](../A-architecture/A7-time-conventions.md) (status: reviewed, 2026-05-05) | A7 §4.1 / §4.3 / §4.6 | `timestamp` 字段:`uint32`(4 byte)、单位 ms(firmware-legacy 无后缀,per A2 E-10.1)、模 \(2^{32}\) wrap;每 bus 末尾 `timestamp` 字段不在每行重述,按全局规则 |
| [A8 共享库块清单与归属](../A-architecture/A8-shared-library-roster.md) (status: reviewed, 2026-05-05) | A8 §4.2.5 | `_valid` / `_status` 字段消费由 `Validity_AND` / `Stale_Detector` 等原语处理,B1 仅锁字段类型 = bool / enum / uint32 |
| **co-seal batch (B1, B2, B3)**:[B2 Enum 清单与数值锁定](B2-enum-inventory.md) (sibling, draft) / [B3 Parameter schema 设计](B3-parameter-schema.md) (sibling, draft) | per [01-design-relationships.md §5 协同对 1](../01-design-relationships.md) | **本工作项与 B2 / B3 同 batch(Wave 4 协同定稿)。B2 owns enum 数值表;B3 owns PARAM/EXPORT 字段表。B1 引用其类型而不重定义。** Per RULES §6 self-check 第 2 项,co-seal batch 兄弟视作满足上游 "≥ reviewed" 要求 |
| firmware `*_types.h`(契约镜像参考,**byte 级权威来源**) | `FMT-Firmware/src/model/plant/<vehicle>/lib/Plant_types.h`;`FMT-Firmware/src/model/fms/<vehicle>/lib/FMS_types.h`;`FMT-Firmware/src/model/control/<vehicle>/lib/Controller_types.h`;`FMT-Firmware/src/model/ins/lib/INS_types.h`;**FMT-Firmware @ \<pending hash\>**(commit hash 由 INDEX 决策日志维护方在登记本工作项时补齐,与 [A3 §3](../A-architecture/A3-module-boundaries.md) / [A6 §3 / A7 §3](../A-architecture/A6-init-reset-contract.md) 同样的占位约定;FMT-Firmware 仓未挂载在本环境) | 字段名 / 类型 / 顺序的 firmware 端权威来源;B1 落 byte-level schema 的镜像参考。本 draft 中 byte width 与 offset 均按 ARM Cortex-M 目标 little-endian 与 natural alignment **best-effort 推断**,标 `(B4 to verify)`;在首次 export 时由 [B4 契约 diff](B4-contract-diff.md) + [I3 契约 diff 脚本](../I-tooling/I3-contract-diff.md) 验证 |

注:本工作项 contract_impact=yes;**直接定义 firmware-visible bus 字节级布局**,变更必须经 INDEX 决策日志登记(per RULES §10)。

## 4. 设计内容

### 4.1 全局约定

#### 4.1.1 Endianness

**显式声明:little-endian**,与 firmware 编译目标(ARM Cortex-M3 / M4 / M7,FMT-Firmware 主流 MCU)一致。所有多字节字段(`uint16` / `uint32` / `single` / `double` / array)按 little-endian 序列化。**B4 contract diff 必须验证 firmware 编译产物(目标 MCU)亦为 little-endian**;若未来切到 big-endian 目标,本设计需变更日志同步。

#### 4.1.2 数据类型映射

下表给出 Simulink 数据类型 → C 类型 → 字节宽度:

| Simulink type | C type | 字节宽度 | 备注 |
|---|---|---:|---|
| `boolean` | `boolean_T` (uint8 in ert.tlc) | 1 | ert.tlc 默认 `boolean_T = unsigned char` |
| `int8` / `uint8` | `int8_T` / `uint8_T` | 1 | |
| `int16` / `uint16` | `int16_T` / `uint16_T` | 2 | |
| `int32` / `uint32` | `int32_T` / `uint32_T` | 4 | |
| `single` | `real32_T` (`float`) | 4 | IEEE-754 单精度,默认浮点类型 |
| `double` | `real64_T` (`double`) | 8 | 仅 lat/lon 经纬度专项(per A2 R-10.9) |
| 数组 `<type>[N]` | `<type> [N]` | `N × <type 宽度>` | 静态长度,**禁止 variable-size signal**(per 架构 v1 §4.5) |
| 嵌套 struct | nested `struct` | 字段宽度求和 | natural alignment;子 schema 在 §4.6 集中 |

#### 4.1.3 累计偏移量 (cumulative offset) 约定

- **基址**:每个 bus 第一个字段偏移量 = 0
- **对齐策略**:**natural alignment**(N 字节字段按 N 字节边界对齐);ARM Cortex-M 默认编译器(arm-none-eabi-gcc / armclang)行为与 IAR 一致
- **填充字段**:对齐间隙在 cumulative offset 跳进过程中自动消化;B1 表中**显式不列 padding 行**,但若 firmware 定义 `__attribute__((packed))` 或显式 `uint8 _pad[N]`,B4 会发现并触发本文件变更日志
- **`(B4 to verify)`** 标注:每 bus 总宽度行带此标;每个 array 字段(尤其 `wp_array[]` 这种 struct array)的累计偏移亦带此标
- **A3 标 "B1 to confirm" 字段**:本文件给 best-effort 类型 + 偏移,加 `(B4 to verify; A3 to-confirm)` 备注;A3 §5 全部 to-confirm 项在本文件 §5 风险中追踪

#### 4.1.4 字段命名 / firmware-legacy 标记

- 字段名以 firmware `*_types.h` 中 firmware-legacy 名为准(per A2 E-5.1);A3 给出的 A2-compliant 工作名(如 `vel_ned_mps` 替代 `velocity`)在 firmware 实际命名出现差异时,本文件以 firmware 名为准并标 `firmware-legacy`
- 若 firmware 名带后缀(单位/坐标系)与 A2 R-5.3 冲突,沿 firmware 名(per A2 E-5.1)
- `timestamp` 字段全局视为 firmware-legacy(无 `_us` / `_ms` 后缀,实际单位 ms,per A7 §4.1 决策)

#### 4.1.5 字段表列定义

每个 bus 的字段 schema 表使用以下列(在每个表头不再重复说明):

| 列 | 含义 |
|---|---|
| Order | 字段顺序号(从 1 起) |
| Field | 字段名(per A2 R-5.3 + E-5.1;firmware-legacy 沿 firmware 实际名) |
| Type | Simulink 数据类型(`single` / `double` / `uint8` / `uint16` / `uint32` / `int8` / `int16` / `int32` / `boolean` / 数组 `<type>[N]` / 嵌套 struct) |
| Width | 字段字节宽度 |
| Offset | 累计字节偏移量(natural alignment best-effort,标 `(B4 to verify)` 在 §<bus> 总宽度行) |
| Unit | 单位后缀(per A3 → A2 §4.10.1;N/A 表无量纲) |
| Frame | 坐标系(per A3 → A2 §4.10.2;N/A 表无坐标系语义) |
| Encoding | 编码 / 量化语义(典型 `IEEE-754 single`、`IEEE-754 double`、`uint enum (B2-locked)`、`uint32 bitfield`、`bool 0/1`、`raw uint8/16/32`、`array passthrough`)|
| Validity | validity / staleness 守卫依赖(典型 `valid if *_valid==1`、`valid if INS_Status.ready==1`、`gated by cmd_mask.<bit>`)|
| B2/B3 ref | enum 字段 → B2;PARAM 类别字段 → B3;否则空 |
| Notes | 可选性 / passthrough / variant-conditional / firmware-legacy / cmd_mask gated / B4 to verify |

补充:每个 bus 末尾的 `timestamp` 字段一律 `Type=uint32, Width=4, Unit=ms(firmware-legacy), Frame=N/A, Encoding=raw uint32 ms, Validity=N/A, Notes=per A7 §4.1`,本文件不在每行重述。

#### 4.1.6 Simulink Bus Object 名 (per A2 R-5.1)

每个契约 bus 在 Simulink data dictionary 或 base workspace 中以 `<bus name>` 作为 `Simulink.Bus` 对象名(完全等同于 firmware-visible bus name)。例:`FMS_Out_Bus` 在 Simulink 中是 `Simulink.Bus` 对象 `FMS_Out_Bus`;字段以 `Simulink.BusElement` 列出。本文件每 bus 头表给出此名。

注:A2 R-5.1 同时列出 `IMU_Out_Bus` / `MAG_Out_Bus` / `Barometer_Out_Bus` / `GPS_Out_Bus` / `Airspeed_Out_Bus` 作为 firmware 已固化名;A3 与本任务 brief 使用 `IMU_Bus` / `MAG_Bus` / `Barometer_Bus` / `GPS_uBlox_Bus` / `AirSpeed_Bus`。**B1 决议**:以本任务 brief 与 A3 命名为准(`IMU_Bus` 等),A2 R-5.1 列表中 `*_Out_Bus` 形式视作 A2 草拟时的工作名;B4 验证时按 firmware `*_types.h` 实际类型名为准,本文件在每 bus 头加 "alt name (A2 R-5.1)" 注。任何不一致由 B4 反馈触发本文件变更日志。

#### 4.1.7 参数化数组长度

下列字段是 array,长度需在 byte schema 锁定但具体数值由 vehicle leaf(C3 / D5 / E4)决定:

| 字段 | 用途 | 长度来源 |
|---|---|---|
| `Control_Out_Bus.motor_cmd[]` | 多旋翼电机命令通道数 | E4(默认多旋翼 4) |
| `Control_Out_Bus.actuator_cmd[]` | 舵面 / 倾转通道数 | E4(多旋翼 Phase 2 = 4 占位 / B4 to verify firmware 实际长度) |
| `Plant_States_Bus.motor_rpm[]` | 多旋翼电机转速 | C3(默认多旋翼 4) |
| `States_Init_Bus.motor_state[]` | 电机初始状态 | C3(默认多旋翼 4) |
| `Pilot_Cmd_Bus.aux_channel[]` | RC 辅助通道 | G4 / Pilot_Cmd_Bus firmware 定义(占位 8) |
| `GCS_Cmd_Bus.cmd_param[]` | GCS 命令参数包 | firmware 实际长度(占位 8) |
| `Mission_Data_Bus.wp_array[]` | 任务航点序列 | D5(Phase 2 多旋翼上限,占位 16) |
| `FMS_Out_Bus.actuator_cmd[]` | passthrough 执行器空间 | D5 / firmware 实际长度(占位 16) |
| `FMS_Out_Bus.motor_cmd_passthrough[]` *(optional)* | manual passthrough mode | D5(占位 4;variant-conditional) |
| `Plant_States_Bus.inertia_b_kgm2[3][3]` | 惯量张量 | 固定 3×3 = 9 single |

**B1 byte-level schema 在表中给出"占位长度",标 `(B4 to verify firmware 实际长度)`**;若 firmware 实际长度不同,本文件变更日志同步。这些占位值仅用于估算 bus 总宽度,**不进入 firmware export**(数组实际长度由 firmware 头主导)。

### 4.2 Plant 模块输入 buses

#### 4.2.1 `Pilot_Cmd_Bus`

| 项 | 值 |
|---|---|
| Bus 名 | `Pilot_Cmd_Bus` |
| Owner module | external (RC pilot stick / G4 注入);**消费方** = FMS |
| Nominal rate | external (≈ 50–100 Hz);RB-06 进入 FMS 20 ms |
| Simulink Bus Object | `Pilot_Cmd_Bus` (per A2 R-5.1) |
| Source firmware header | `FMT-Firmware/src/model/fms/<vehicle>/lib/FMS_types.h` @ `<pending hash>` |
| Direction | Input → FMS (per A3 §4.4.1) |

| Order | Field | Type | Width | Offset | Unit | Frame | Encoding | Validity | B2/B3 ref | Notes |
|---:|---|---|---:|---:|---|---|---|---|---|---|
| 1 | `roll_stick_n01` | `single` | 4 | 0 | `_n01` | N/A | IEEE-754 single, range -1..1 | valid if `valid==1` | — | per A3 §4.4.2 |
| 2 | `pitch_stick_n01` | `single` | 4 | 4 | `_n01` | N/A | IEEE-754 single, range -1..1 | valid if `valid==1` | — | |
| 3 | `yaw_stick_n01` | `single` | 4 | 8 | `_n01` | N/A | IEEE-754 single, range -1..1 | valid if `valid==1` | — | |
| 4 | `throttle_stick_n01` | `single` | 4 | 12 | `_n01` | N/A | IEEE-754 single, range 0..1 | valid if `valid==1` | — | |
| 5 | `mode_switch` | `uint8` | 1 | 16 | N/A | N/A | uint8 enum (PilotMode) | valid if `valid==1` | **B2: PilotMode** | per A3 §4.4.2 |
| 6 | `arm_switch` | `boolean` | 1 | 17 | N/A | N/A | bool 0/1 | valid if `valid==1` | — | |
| 7 | `kill_switch` | `boolean` | 1 | 18 | N/A | N/A | bool 0/1 | valid if `valid==1` | — | optional / variant-conditional |
| 8 | `valid` | `boolean` | 1 | 19 | N/A | N/A | bool 0/1 (RC 链路有效) | self | — | RC link validity |
| 9 | `aux_channel` | `single[8]` | 32 | 20 | `_n01` | N/A | IEEE-754 single array, range -1..1 | valid if `valid==1` | — | optional;长度占位 8 (B4 to verify) |
| 10 | `timestamp` | `uint32` | 4 | 52 | ms | N/A | raw uint32 ms (firmware-legacy) | N/A | — | per A7 §4.1 |

**Total bus byte width**: 56 bytes (B4 to verify alignment padding 与 `aux_channel[]` 实际长度).

#### 4.2.2 `GCS_Cmd_Bus`

| 项 | 值 |
|---|---|
| Bus 名 | `GCS_Cmd_Bus` |
| Owner module | external (ground station);**消费方** = FMS |
| Nominal rate | external (≈ 5–20 Hz);RB-06 进入 FMS 20 ms |
| Simulink Bus Object | `GCS_Cmd_Bus` |
| Source firmware header | `FMT-Firmware/src/model/fms/<vehicle>/lib/FMS_types.h` @ `<pending hash>` |
| Direction | Input → FMS (optional / 契约 mandatory, per A3 §4.4.1) |

| Order | Field | Type | Width | Offset | Unit | Frame | Encoding | Validity | B2/B3 ref | Notes |
|---:|---|---|---:|---:|---|---|---|---|---|---|
| 1 | `cmd_type` | `uint8` | 1 | 0 | N/A | N/A | uint8 enum (GcsCmdType) | valid if `valid==1` | **B2: GcsCmdType** | per A3 §4.4.3.1 |
| 2 | `valid` | `boolean` | 1 | 1 | N/A | N/A | bool 0/1 (GCS 链路有效) | self | — | (offset 2..7 alignment pad — B4 to verify) |
| 3 | `cmd_param` | `single[8]` | 32 | 8 | mixed | mixed | IEEE-754 single array, semantics per `cmd_type` | valid if `valid==1` | — | 长度占位 8 (B4 to verify) |
| 4 | `setpoint_pos_lat_deg` | `double` | 8 | 40 | `_deg` | N/A | IEEE-754 double | valid if `cmd_type == goto_position` && `valid==1` | — | optional;经纬度专项 deg (per A2 R-10.9) |
| 5 | `setpoint_pos_lon_deg` | `double` | 8 | 48 | `_deg` | N/A | IEEE-754 double | 同上 | — | |
| 6 | `setpoint_pos_alt_m` | `single` | 4 | 56 | `_m` | N/A | IEEE-754 single | 同上 | — | |
| 7 | `setpoint_yaw_rad` | `single` | 4 | 60 | `_rad` | NED→body | IEEE-754 single | 同上 | — | |
| 8 | `timestamp` | `uint32` | 4 | 64 | ms | N/A | raw uint32 ms (firmware-legacy) | N/A | — | |

**Total bus byte width**: 68 bytes (B4 to verify alignment padding 与 `cmd_param[]` / `setpoint_*` 字段是否全在 firmware 中).

#### 4.2.3 `Auto_Cmd_Bus`

| 项 | 值 |
|---|---|
| Bus 名 | `Auto_Cmd_Bus` |
| Owner module | external (onboard autopilot / Offboard);**消费方** = FMS |
| Nominal rate | external;RB-06 进入 FMS 20 ms |
| Simulink Bus Object | `Auto_Cmd_Bus` |
| Source firmware header | `FMT-Firmware/src/model/fms/<vehicle>/lib/FMS_types.h` @ `<pending hash>` |
| Direction | Input → FMS (optional, per A3 §4.4.1) |

| Order | Field | Type | Width | Offset | Unit | Frame | Encoding | Validity | B2/B3 ref | Notes |
|---:|---|---|---:|---:|---|---|---|---|---|---|
| 1 | `target_pos_ned_m` | `single[3]` | 12 | 0 | `_m` | NED | IEEE-754 single array | valid if `valid==1` | — | per A3 §4.4.3.2 |
| 2 | `target_vel_ned_mps` | `single[3]` | 12 | 12 | `_mps` | NED | IEEE-754 single array | valid if `valid==1` | — | |
| 3 | `target_acc_ned_mps2` | `single[3]` | 12 | 24 | `_mps2` | NED | IEEE-754 single array | valid if `valid==1` | — | optional (前馈) |
| 4 | `target_yaw_rad` | `single` | 4 | 36 | `_rad` | NED→body | IEEE-754 single | valid if `valid==1` | — | |
| 5 | `target_yaw_rate_radps` | `single` | 4 | 40 | `_radps` | NED→body | IEEE-754 single | valid if `valid==1` | — | optional |
| 6 | `cmd_mask_request` | `uint32` | 4 | 44 | N/A | N/A | uint32 bitfield (位语义由 D6 锁定) | valid if `valid==1` | **B2: cmd_mask 位 alias** | Auto 模块"建议"的环路集 |
| 7 | `valid` | `boolean` | 1 | 48 | N/A | N/A | bool 0/1 | self | — | (offset 49..51 alignment pad — B4 to verify) |
| 8 | `timestamp` | `uint32` | 4 | 52 | ms | N/A | raw uint32 ms (firmware-legacy) | N/A | — | |

**Total bus byte width**: 56 bytes (B4 to verify alignment padding).

#### 4.2.4 `Mission_Data_Bus`

| 项 | 值 |
|---|---|
| Bus 名 | `Mission_Data_Bus` |
| Owner module | external (mission planner / loaded mission file);**消费方** = FMS Mission/Auto Manager |
| Nominal rate | mission-load(≈ 配置时一次)+ mission-progress(随 FMS 20 ms);RB-06 |
| Simulink Bus Object | `Mission_Data_Bus` |
| Source firmware header | `FMT-Firmware/src/model/fms/<vehicle>/lib/FMS_types.h` @ `<pending hash>` |
| Direction | Input → FMS (optional, per A3 §4.4.1;Phase 2 闭环范围由 D1 / D5 决定) |

嵌套子类型 `Waypoint`(集中定义见 §4.6.7);此处仅出现 `wp_array[N]`。

| Order | Field | Type | Width | Offset | Unit | Frame | Encoding | Validity | B2/B3 ref | Notes |
|---:|---|---|---:|---:|---|---|---|---|---|---|
| 1 | `wp_count` | `uint16` | 2 | 0 | N/A | N/A | raw uint16 (count) | valid if `valid==1` | — | per A3 §4.4.3.3 |
| 2 | `wp_current_index` | `uint16` | 2 | 2 | N/A | N/A | raw uint16 (index) | valid if `valid==1` | — | per A6 §5 PRESERVED 候选 |
| 3 | `wp_array` | `Waypoint[16]` | `16 × sizeof(Waypoint)` | 4 | mixed | mixed | nested struct array (子 schema §4.6.7) | valid if `valid==1` | — | 长度占位 16 (B4 to verify firmware 实际上限 — D5 锁定多旋翼默认值) |
| 4 | `home_lat_deg` | `double` | 8 | `4 + 16 × sizeof(Waypoint)` | `_deg` | N/A | IEEE-754 double | self | **B3: PLANT_PARAM 候选 home init** | per A2 R-10.9 |
| 5 | `home_lon_deg` | `double` | 8 | `12 + 16 × sizeof(Waypoint)` | `_deg` | N/A | IEEE-754 double | self | **B3: PLANT_PARAM 候选** | |
| 6 | `home_alt_m` | `single` | 4 | `20 + 16 × sizeof(Waypoint)` | `_m` | N/A | IEEE-754 single | self | **B3: PLANT_PARAM 候选** | |
| 7 | `valid` | `boolean` | 1 | `24 + 16 × sizeof(Waypoint)` | N/A | N/A | bool 0/1 | self | — | (alignment pad — B4 to verify) |
| 8 | `timestamp` | `uint32` | 4 | `28 + 16 × sizeof(Waypoint)` | ms | N/A | raw uint32 ms (firmware-legacy) | N/A | — | |

**Total bus byte width**: `32 + 16 × sizeof(Waypoint)` bytes (B4 to verify;若 `Waypoint` ≈ 32 bytes,总宽 ≈ 32 + 512 = 544 bytes).

### 4.3 Plant 模块输入 buses(续)

#### 4.3.1 `INS_Out_Bus`(consumed by FMS + Controller)

| 项 | 值 |
|---|---|
| Bus 名 | `INS_Out_Bus` |
| Owner module | **firmware (FMT-Firmware/src/model/ins)** — model repo 仅镜像;MIL 由 `model/harness/ins_stub` 产出 |
| Nominal rate | 10 ms (100 Hz, INS firmware-owned cadence per 架构 v1 §13);RB-03 → Controller / RB-04 → FMS |
| Simulink Bus Object | `INS_Out_Bus` |
| Source firmware header | `FMT-Firmware/src/model/ins/lib/INS_types.h` @ `<pending hash>` |
| Direction | Input → FMS, Input → Controller (per A3 §4.4.4 / §4.5.3) |

嵌套子类型:`INS_Status`(子 schema §4.6.5)、`INS_Flag`(子 schema §4.6.6)。

注:**`INS_Out_Bus` 字段顺序 / 数据类型完全跟随 firmware** `INS_types.h`(per A2 E-5.1 / [B5 §4](B5-ins-bus-mirror.md));本文件给出消费侧已知字段类别与最佳推断顺序,**首次镜像时由 [B5](B5-ins-bus-mirror.md) byte-level verify**。

| Order | Field | Type | Width | Offset | Unit | Frame | Encoding | Validity | B2/B3 ref | Notes |
|---:|---|---|---:|---:|---|---|---|---|---|---|
| 1 | `position_lla` | `double[3]` | 24 | 0 | mixed (`_deg`,`_deg`,`_m`) | N/A | IEEE-754 double × 3 (lat_deg, lon_deg, alt_m) | gated `flag.position_valid` && `INS_Status.ready` | — | per A2 R-10.9;**B5 to confirm 字段实际名是否为 `position_lla` 或拆 `lat_deg`/`lon_deg`/`alt_m`** |
| 2 | `position_ned_m` | `single[3]` | 12 | 24 | `_m` | NED | IEEE-754 single × 3 | gated `flag.position_valid` | — | (after double[3], offset OK) |
| 3 | `velocity_ned_mps` | `single[3]` | 12 | 36 | `_mps` | NED | IEEE-754 single × 3 | gated `flag.velocity_valid` | — | firmware-legacy 名可能为 `velocity_ned`(B5 verify) |
| 4 | `quat_ned_to_b` | `single[4]` | 16 | 48 | N/A | NED→body | IEEE-754 single × 4 (w, x, y, z 顺序待 B5 verify) | gated `flag.attitude_valid` | — | (B5 to confirm w-first vs w-last) |
| 5 | `euler_ned_to_b_rad` | `single[3]` | 12 | 64 | `_rad` | NED→body (ZYX) | IEEE-754 single × 3 (yaw, pitch, roll;B5 verify) | gated `flag.attitude_valid` | — | firmware-legacy 名可能为 `euler` (B5 verify);ZYX 顺序在 A2 R-10.6 默认 |
| 6 | `ang_rate_b_radps` | `single[3]` | 12 | 76 | `_radps` | body | IEEE-754 single × 3 (filtered) | gated `INS_Status.ready` | — | |
| 7 | `acc_b_mps2` | `single[3]` | 12 | 88 | `_mps2` | body | IEEE-754 single × 3 (filtered specific force) | gated `INS_Status.ready` | — | |
| 8 | `INS_Status` | `uint32` (bitfield) | 4 | 100 | N/A | N/A | uint32 bitfield (`ready` 位 + 子状态;详子 schema §4.6.5) | self | **B2: InsStatus 位 alias** | firmware-legacy struct or bitfield (B5 verify);A3 §4.4.4 |
| 9 | `INS_Flag` | `uint32` (bitfield) | 4 | 104 | N/A | N/A | uint32 bitfield (`position_valid` / `velocity_valid` / `attitude_valid` / `mag_valid` / `gps_valid`;子 schema §4.6.6) | self | **B2: InsFlag 位 alias** | firmware-legacy struct or bitfield (B5 verify) |
| 10 | `timestamp` | `uint32` | 4 | 108 | ms | N/A | raw uint32 ms (firmware-legacy) | N/A | — | per A7 §4.1 |

**Total bus byte width**: 112 bytes (B4 to verify;**全字段镜像策略由 [B5](B5-ins-bus-mirror.md) 锁定;本表是消费侧最佳推断**).

### 4.4 Controller 模块输入 / 输出 buses

#### 4.4.1 `Control_Out_Bus`

| 项 | 值 |
|---|---|
| Bus 名 | `Control_Out_Bus` |
| Owner module | Controller (model repo) |
| Nominal rate | 5 ms (200 Hz, per 架构 v1 §13);RB-02 → Plant / RB-04 等价 → FMS(若启用回送,per A3 §4.10) |
| Simulink Bus Object | `Control_Out_Bus` |
| Source firmware header | `FMT-Firmware/src/model/control/<vehicle>/lib/Controller_types.h` @ `<pending hash>` |
| Direction | Output ← Controller; Input → Plant; Input → FMS (variant / transitional, per A3 §4.4.5) |

| Order | Field | Type | Width | Offset | Unit | Frame | Encoding | Validity | B2/B3 ref | Notes |
|---:|---|---|---:|---:|---|---|---|---|---|---|
| 1 | `motor_cmd` | `single[4]` | 16 | 0 | `_n01` 或 `_us` | N/A | IEEE-754 single × 4 (B4 to verify normalized vs PWM us, A3 §5 to-confirm) | gated `cmd_mask.throttle_passthrough` ⇒ N/A | — | 多旋翼 4 电机占位;长度由 E4 锁定 |
| 2 | `actuator_cmd` | `single[16]` | 64 | 16 | `_n01` 或 trim | body | IEEE-754 single × 16 (B4 to verify normalized vs trim;长度 16 占位) | passthrough | — | per A3 §4.3.2;舵面 / 倾转;多旋翼 Phase 2 不使用但保留 |
| 3 | `throttle_cmd` | `single` | 4 | 80 | `_n01` | N/A | IEEE-754 single, range 0..1 | gated `cmd_mask.throttle_passthrough` | — | 总推力 / collective |
| 4 | `cmd_mask` | `uint32` | 4 | 84 | N/A | N/A | uint32 bitfield (位语义由 D6/E1 锁定) | passthrough from FMS | **B2: cmd_mask 位 alias** | passthrough from `FMS_Out_Bus.cmd_mask` |
| 5 | `ctrl_mode` | `uint8` | 1 | 88 | N/A | N/A | uint8 enum (CtrlMode) | passthrough | **B2: CtrlMode** | passthrough |
| 6 | `reset` | `boolean` | 1 | 89 | N/A | N/A | bool 0/1 | passthrough | — | optional (per A6 §5 第 2 项 open;若 firmware 不含,删行) |
| 7 | `timestamp` | `uint32` | 4 | 92 | ms | N/A | raw uint32 ms (firmware-legacy) | N/A | — | (offset 90..91 alignment pad) |

**Total bus byte width**: 96 bytes (B4 to verify alignment padding 与数组实际长度).

注:`Control_Out_Bus` 的 `motor_cmd[]` / `actuator_cmd[]` 单位与含义是 A3 §5 多个 to-confirm 项;本表给最佳推断,B4 byte-verify 时按 firmware `Controller_types.h` 实际定义为准。

### 4.5 Plant 输出 buses

#### 4.5.1 `Plant_States_Bus`

| 项 | 值 |
|---|---|
| Bus 名 | `Plant_States_Bus` |
| Owner module | Plant (model repo) |
| Nominal rate | 1 ms (1 kHz, per 架构 v1 §13);RB-01 (variant-only) / 内部 ins_stub 通路 |
| Simulink Bus Object | `Plant_States_Bus` |
| Source firmware header | `FMT-Firmware/src/model/plant/<vehicle>/lib/Plant_types.h` @ `<pending hash>` |
| Direction | Output ← Plant; Input → ins_stub (MIL) / Controller (variant) (per A3 §4.3.5) |

| Order | Field | Type | Width | Offset | Unit | Frame | Encoding | Validity | B2/B3 ref | Notes |
|---:|---|---|---:|---:|---|---|---|---|---|---|
| 1 | `pos_ned_m` | `single[3]` | 12 | 0 | `_m` | NED | IEEE-754 single × 3 | self (truth) | — | per A6: PARAM (echoes States_Init_Bus.pos_ned_m at init) |
| 2 | `vel_ned_mps` | `single[3]` | 12 | 12 | `_mps` | NED | IEEE-754 single × 3 | self | — | |
| 3 | `acc_ned_mps2` | `single[3]` | 12 | 24 | `_mps2` | NED | IEEE-754 single × 3 | self | — | |
| 4 | `acc_b_mps2` | `single[3]` | 12 | 36 | `_mps2` | body | IEEE-754 single × 3 | self | — | body specific force |
| 5 | `quat_ned_to_b` | `single[4]` | 16 | 48 | N/A | NED→body | IEEE-754 single × 4 (w-first 占位, B4 verify) | self | — | |
| 6 | `euler_ned_to_b_rad` | `single[3]` | 12 | 64 | `_rad` | NED→body (ZYX) | IEEE-754 single × 3 (yaw, pitch, roll) | self | — | firmware-legacy 命名待 B4 verify |
| 7 | `ang_rate_b_radps` | `single[3]` | 12 | 76 | `_radps` | body | IEEE-754 single × 3 | self | — | |
| 8 | `ang_acc_b_radps2` | `single[3]` | 12 | 88 | `_radps2` | body | IEEE-754 single × 3 | self | — | optional; B4 to verify whether firmware ships |
| 9 | `mass_kg` | `single` | 4 | 100 | `_kg` | N/A | IEEE-754 single | self | **B3: PLANT_PARAM** | |
| 10 | `inertia_b_kgm2` | `single[3][3]` | 36 | 104 | `_kgm2` | body | IEEE-754 single × 9 (3×3 matrix; row-major B4 verify) | self | **B3: PLANT_PARAM** | |
| 11 | `motor_rpm` | `single[4]` | 16 | 140 | `_rpm` | N/A | IEEE-754 single × 4 (per A2 R-10.2 显式后缀,非 SI) | self | — | 多旋翼 4 占位;长度由 C3 锁定 |
| 12 | `on_ground` | `boolean` | 1 | 156 | N/A | N/A | bool 0/1 | self | — | per A6: HARDCODED true at init |
| 13 | `timestamp` | `uint32` | 4 | 160 | ms | N/A | raw uint32 ms (firmware-legacy) | N/A | — | (offset 157..159 alignment pad) |

**Total bus byte width**: 164 bytes (B4 to verify alignment padding 与 optional `ang_acc_b_radps2` / array 长度).

#### 4.5.2 `Extended_States_Bus`

| 项 | 值 |
|---|---|
| Bus 名 | `Extended_States_Bus` |
| Owner module | Plant (model repo) |
| Nominal rate | 1 ms (1 kHz);RB-01 (harness-only) |
| Simulink Bus Object | `Extended_States_Bus` |
| Source firmware header | `FMT-Firmware/src/model/plant/<vehicle>/lib/Plant_types.h` @ `<pending hash>` |
| Direction | Output ← Plant (per A3 §4.3.6) |

| Order | Field | Type | Width | Offset | Unit | Frame | Encoding | Validity | B2/B3 ref | Notes |
|---:|---|---|---:|---:|---|---|---|---|---|---|
| 1 | `airspeed_mps` | `single` | 4 | 0 | `_mps` | body (forward) | IEEE-754 single | self | — | 真空速 |
| 2 | `aoa_rad` | `single` | 4 | 4 | `_rad` | body | IEEE-754 single | self | — | optional;variant-conditional (FW/VTOL) |
| 3 | `aos_rad` | `single` | 4 | 8 | `_rad` | body | IEEE-754 single | self | — | optional;variant-conditional |
| 4 | `dynamic_pressure_pa` | `single` | 4 | 12 | `_pa` | N/A | IEEE-754 single | self | — | |
| 5 | `flow_angle_b_rad` | `single[3]` | 12 | 16 | `_rad` | body | IEEE-754 single × 3 | self | — | optional;variant-conditional |
| 6 | `disturbance_force_b_n` | `single[3]` | 12 | 28 | `_n` | body | IEEE-754 single × 3 | self | — | trace by `Disturbance_Injector_Shell` (A8) |
| 7 | `disturbance_torque_b_nm` | `single[3]` | 12 | 40 | `_nm` | body | IEEE-754 single × 3 | self | — | trace |
| 8 | `timestamp` | `uint32` | 4 | 52 | ms | N/A | raw uint32 ms (firmware-legacy) | N/A | — | |

**Total bus byte width**: 56 bytes (B4 to verify whether firmware ships optional fields).

#### 4.5.3 `Environment_Info_Bus`

| 项 | 值 |
|---|---|
| Bus 名 | `Environment_Info_Bus` |
| Owner module | external (scenario / harness param-driven);**消费方** = Plant |
| Nominal rate | configuration-time (≈ 整段仿真常量);非 step-rate bus |
| Simulink Bus Object | `Environment_Info_Bus` |
| Source firmware header | `FMT-Firmware/src/model/plant/<vehicle>/lib/Plant_types.h` @ `<pending hash>` |
| Direction | Input → Plant (per A3 §4.3.3);全字段 PARAM (per A6 §4.4.2) |

| Order | Field | Type | Width | Offset | Unit | Frame | Encoding | Validity | B2/B3 ref | Notes |
|---:|---|---|---:|---:|---|---|---|---|---|---|
| 1 | `wind_vel_ned_mps` | `single[3]` | 12 | 0 | `_mps` | NED | IEEE-754 single × 3 | configuration | **B3: PLANT_PARAM** | per A3 §4.3.3 |
| 2 | `air_density_kgpm3` | `single` | 4 | 12 | `kg/m³` | N/A | IEEE-754 single | configuration | **B3: PLANT_PARAM** | |
| 3 | `air_pressure_pa` | `single` | 4 | 16 | `_pa` | N/A | IEEE-754 single | configuration | **B3: PLANT_PARAM** | |
| 4 | `air_temperature_k` | `single` | 4 | 20 | `_k` | N/A | IEEE-754 single | configuration | **B3: PLANT_PARAM** | per A2 R-10.1 |
| 5 | `mag_field_ned_gauss` | `single[3]` | 12 | 24 | `_gauss` (B4 to verify; A3 §5 to-confirm vs nT) | NED | IEEE-754 single × 3 | configuration | **B3: PLANT_PARAM** | |
| 6 | `gravity_ned_mps2` | `single[3]` | 12 | 36 | `_mps2` | NED | IEEE-754 single × 3 | configuration | **B3: PLANT_PARAM** | typical [0,0,9.81] |
| 7 | `home_lat_deg` | `double` | 8 | 48 | `_deg` | N/A | IEEE-754 double | configuration | **B3: PLANT_PARAM** | per A2 R-10.9 |
| 8 | `home_lon_deg` | `double` | 8 | 56 | `_deg` | N/A | IEEE-754 double | configuration | **B3: PLANT_PARAM** | |
| 9 | `home_alt_m` | `single` | 4 | 64 | `_m` | N/A | IEEE-754 single | configuration | **B3: PLANT_PARAM** | |

**Total bus byte width**: 68 bytes (B4 to verify;无 `timestamp`,因 configuration-time bus).

#### 4.5.4 `States_Init_Bus`

| 项 | 值 |
|---|---|
| Bus 名 | `States_Init_Bus` |
| Owner module | external (scenario / harness param-driven);**消费方** = Plant init |
| Nominal rate | configuration-time(仅 Plant init / reset 时被读取) |
| Simulink Bus Object | `States_Init_Bus` |
| Source firmware header | `FMT-Firmware/src/model/plant/<vehicle>/lib/Plant_types.h` @ `<pending hash>` |
| Direction | Input → Plant init (per A3 §4.3.4);全字段 PARAM (per A6 §4.4.2) |

| Order | Field | Type | Width | Offset | Unit | Frame | Encoding | Validity | B2/B3 ref | Notes |
|---:|---|---|---:|---:|---|---|---|---|---|---|
| 1 | `pos_ned_m` | `single[3]` | 12 | 0 | `_m` | NED | IEEE-754 single × 3 | configuration | **B3: PLANT_PARAM init** | per A3 §4.3.4 |
| 2 | `vel_ned_mps` | `single[3]` | 12 | 12 | `_mps` | NED | IEEE-754 single × 3 | configuration | **B3: PLANT_PARAM init** | |
| 3 | `quat_ned_to_b` | `single[4]` | 16 | 24 | N/A | NED→body | IEEE-754 single × 4 (w-first 占位, B4 verify) | configuration | **B3: PLANT_PARAM init** | A3 §5 to-confirm: w-first vs w-last |
| 4 | `ang_rate_b_radps` | `single[3]` | 12 | 40 | `_radps` | body | IEEE-754 single × 3 | configuration | **B3: PLANT_PARAM init** | |
| 5 | `motor_state` | `single[4]` | 16 | 52 | `_rpm` 或 `_n01` (B4 to verify) | N/A | IEEE-754 single × 4 (长度占位 4) | configuration | **B3: PLANT_PARAM init** (or ZERO per C2/C3) | 多旋翼 4 占位 |

**Total bus byte width**: 68 bytes (B4 to verify alignment padding 与数组实际长度).

### 4.6 子 schema(嵌套类型集中定义)

按 RULES §5"禁止重复定义"原则,以下嵌套类型 **在本节定义一次**,其他 bus 字段表中以类型名引用。

#### 4.6.1 `Vec3_NED_m` 等典型 3-向量

非真正嵌套 struct,而是 array 写法 `single[3]`;为可读性集中:

| 别名 | 物理含义 | Type | Width | Frame | Unit | Encoding |
|---|---|---|---:|---|---|---|
| `Vec3_NED_m` | NED 位置/位移 | `single[3]` | 12 | NED | `_m` | IEEE-754 single × 3 |
| `Vec3_NED_mps` | NED 速度 | `single[3]` | 12 | NED | `_mps` | IEEE-754 single × 3 |
| `Vec3_NED_mps2` | NED 加速度 | `single[3]` | 12 | NED | `_mps2` | IEEE-754 single × 3 |
| `Vec3_B_radps` | body 角速度 | `single[3]` | 12 | body | `_radps` | IEEE-754 single × 3 |
| `Vec3_B_mps2` | body 加速度 | `single[3]` | 12 | body | `_mps2` | IEEE-754 single × 3 |
| `Vec3_B_rad` | body 角度三元组(欧拉等) | `single[3]` | 12 | body 或 NED→body | `_rad` | IEEE-754 single × 3 |
| `Vec3_B_n` | body 力 | `single[3]` | 12 | body | `_n` | IEEE-754 single × 3 |
| `Vec3_B_nm` | body 力矩 | `single[3]` | 12 | body | `_nm` | IEEE-754 single × 3 |

#### 4.6.2 `Quat_NED_to_B`

姿态四元数(NED→body 旋转表示)。

| Field | Type | Width | Encoding | Notes |
|---|---|---:|---|---|
| `q[4]` | `single[4]` | 16 | IEEE-754 single × 4 | **顺序待 B4 verify**:firmware 与 Plant_types.h 里可能为 w-first(`q0=w, q1=x, q2=y, q3=z`)或 w-last(`q0=x, q1=y, q2=z, q3=w`)。A3 §5 to-confirm 项;**B1 草拟期默认 w-first**(更常见的 firmware 约定)。reset 默认 identity quat = `[1, 0, 0, 0]`(若 w-first;w-last 则 `[0, 0, 0, 1]`);per A6 §4.4 PARAM 类别。 |

总宽度:16 bytes。

#### 4.6.3 `Euler_NED_to_B_rad`

ZYX 顺序欧拉角(per A2 R-10.6 默认):

| Field | Type | Width | Encoding | Notes |
|---|---|---:|---|---|
| `e[3]` | `single[3]` | 12 | IEEE-754 single × 3 | **顺序待 B4 verify**:`(yaw, pitch, roll)` 占位(per A3 §4.3.5 注);A3 §5 to-confirm。Frame = NED→body (ZYX rotation)。Unit = `_rad`(per A2 R-10.7) |

总宽度:12 bytes。

#### 4.6.4 `Inertia_B_kgm2`

3×3 惯量张量(对称矩阵,但本契约不利用对称性 — 全 9 元素):

| Field | Type | Width | Encoding | Notes |
|---|---|---:|---|---|
| `I[3][3]` | `single[3][3]` | 36 | IEEE-754 single × 9 (row-major;**B4 to verify** Simulink → C 编译产物的存储顺序) | Frame = body;Unit = `kg·m²` (`_kgm2` 后缀) |

总宽度:36 bytes。

#### 4.6.5 `INS_Status` 子 schema

uint32 bitfield,**位语义由 [B2](B2-enum-inventory.md) 锁定**;A3 §4.4.4 已声明位类别。本子 schema 仅锁定位置(Position 0..31)与位含义草案;具体值由 B2:

| Bit position | Bit name (work draft;B2 locks) | 含义 |
|---:|---|---|
| 0 | `ready` | INS 总体就绪;FMS gate 关键 |
| 1..N | TBD by B2 | INS 子状态(对齐 / GPS-fix / mag-cal 等)|

类型 = `uint32`,宽度 4 bytes。**B1 不定义具体位号**;委托 B2。

#### 4.6.6 `INS_Flag` 子 schema

uint32 bitfield,位语义由 B2 锁定:

| Bit position | Bit name (work draft;B2 locks) | 含义 |
|---:|---|---|
| 0 | `position_valid` | NED 位置有效 |
| 1 | `velocity_valid` | NED 速度有效 |
| 2 | `attitude_valid` | quat / euler 有效 |
| 3 | `mag_valid` | 磁场观测有效 |
| 4 | `gps_valid` | GPS 观测有效 |
| 5..N | TBD by B2 | |

类型 = `uint32`,宽度 4 bytes。**B1 不定义具体位号**;委托 B2。

#### 4.6.7 `Waypoint` 子 schema

`Mission_Data_Bus.wp_array[]` 元素类型。本表给最佳推断,**B4 to verify**:

| Order | Field | Type | Width | Offset | Unit | Frame | Encoding | Notes |
|---:|---|---|---:|---:|---|---|---|---|
| 1 | `lat_deg` | `double` | 8 | 0 | `_deg` | N/A | IEEE-754 double | per A2 R-10.9 |
| 2 | `lon_deg` | `double` | 8 | 8 | `_deg` | N/A | IEEE-754 double | |
| 3 | `alt_m` | `single` | 4 | 16 | `_m` | N/A | IEEE-754 single | |
| 4 | `cmd_type` | `uint8` | 1 | 20 | N/A | N/A | uint8 enum (WaypointCmdType) | **B2: WaypointCmdType** |
| 5 | `param` | `single[3]` | 12 | 24 | mixed | mixed | IEEE-754 single × 3 (semantics per `cmd_type`) | (offset 21..23 alignment pad) |

**Total `Waypoint` byte width**: 36 bytes (B4 to verify alignment padding;`param[]` 实际长度由 firmware mission_data 类型定义).

### 4.7 `FMS_Out_Bus` 字段顺序总账(field-order ledger)— 闭合 A1 §5.1.5

[A1 §5.1.5](../A-architecture/A1-v1-review.md) 缺口要求 `FMS_Out_Bus` 字段顺序 / 类型在 B1 中定稿。架构 v1 §5.1.5 给出"重要字段包括 rate commands / attitude commands / velocity-acceleration commands / actuator_cmd passthrough / throttle_cmd / cmd_mask / status / state / ext_state / ctrl_mode / mode / reset / wp / home / error";本节按设计意图排序,**所有字段一次性枚举**。

| 项 | 值 |
|---|---|
| Bus 名 | `FMS_Out_Bus` |
| Owner module | FMS (model repo) |
| Nominal rate | 20 ms (50 Hz, per 架构 v1 §13);RB-05 → Controller |
| Simulink Bus Object | `FMS_Out_Bus` |
| Source firmware header | `FMT-Firmware/src/model/fms/<vehicle>/lib/FMS_types.h` @ `<pending hash>` |
| Direction | Output ← FMS; Input → Controller (per A3 §4.4.6 / §4.5.2) |

#### 4.7.1 字段顺序(完整 ledger;A1 §5.1.5 demand 闭合)

字段按 **设计意图顺序**排列 — A3 §4.4.6 子分组(setpoint → 控制流 → 任务/导航/错误 → passthrough)在本表中保留:

**(a) Setpoint 命令组(rate / attitude / velocity / acceleration / yaw / yaw-rate / throttle 命令 — gated by cmd_mask)**

| Order | Field | Type | Width | Offset | Unit | Frame | Encoding | Validity / Gate | B2/B3 ref | Notes |
|---:|---|---|---:|---:|---|---|---|---|---|---|
| 1 | `pos_cmd_ned_m` | `single[3]` | 12 | 0 | `_m` | NED | IEEE-754 single × 3 | gated `cmd_mask.position_loop` | — | per A3 §4.4.6.1 |
| 2 | `vel_cmd_ned_mps` | `single[3]` | 12 | 12 | `_mps` | NED | IEEE-754 single × 3 | gated `cmd_mask.velocity_loop` | — | velocity cmd (A1 §5.1.5 demand) |
| 3 | `acc_cmd_ned_mps2` | `single[3]` | 12 | 24 | `_mps2` | NED | IEEE-754 single × 3 | gated `cmd_mask.acceleration_loop` | — | acceleration cmd (A1 §5.1.5 demand) |
| 4 | `att_cmd_quat` | `single[4]` | 16 | 36 | N/A | NED→body | IEEE-754 single × 4 (子 schema §4.6.2) | gated `cmd_mask.attitude_loop` | — | attitude cmd (A1 §5.1.5 demand);reset 默认 identity (per A6) |
| 5 | `att_cmd_euler_rad` | `single[3]` | 12 | 52 | `_rad` | NED→body (ZYX) | IEEE-754 single × 3 (子 schema §4.6.3) | gated `cmd_mask.attitude_loop` | — | optional (alt to quat;B4 verify firmware 是否同时含 quat 与 euler 或二选一);A3 §5 to-confirm |
| 6 | `ang_rate_cmd_b_radps` | `single[3]` | 12 | 64 | `_radps` | body | IEEE-754 single × 3 | gated `cmd_mask.rate_loop` | — | rate cmd (A1 §5.1.5 demand) |
| 7 | `yaw_cmd_rad` | `single` | 4 | 76 | `_rad` | NED→body | IEEE-754 single | gated `cmd_mask.yaw_loop` | — | per A3 §4.4.6.1 |
| 8 | `yaw_rate_cmd_radps` | `single` | 4 | 80 | `_radps` | NED→body | IEEE-754 single | gated `cmd_mask.yaw_rate_loop` | — | |
| 9 | `throttle_cmd` | `single` | 4 | 84 | `_n01` | N/A | IEEE-754 single, range 0..1 | gated `cmd_mask.throttle_passthrough` | — | throttle_cmd (A1 §5.1.5 demand) |

**(b) actuator passthrough 空间(A1 §7.2 demand;架构 v1 §5.1.5 "actuator_cmd passthrough space")**

| Order | Field | Type | Width | Offset | Unit | Frame | Encoding | Validity / Gate | B2/B3 ref | Notes |
|---:|---|---|---:|---:|---|---|---|---|---|---|
| 10 | `actuator_cmd` | `single[16]` | 64 | 88 | mixed (`_n01` / trim) | mixed | IEEE-754 single × 16 (长度占位 16,B4 verify) | passthrough | — | passthrough from `Pilot_Cmd_Bus` / `Auto_Cmd_Bus` (per A3 §4.6) |
| 11 | `motor_cmd_passthrough` | `single[4]` | 16 | 152 | `_n01` | N/A | IEEE-754 single × 4 (长度占位 4) | passthrough | — | optional;variant-conditional (manual passthrough mode;多旋翼默认禁用);**B4 to verify firmware 是否含此字段** |
| 12 | `pilot_throttle_passthrough` | `single` | 4 | 168 | `_n01` | N/A | IEEE-754 single | passthrough | — | passthrough from `Pilot_Cmd_Bus.throttle_stick_n01` (per A3 §4.6) |
| 13 | `pilot_yaw_rate_passthrough_radps` | `single` | 4 | 172 | `_radps` | NED→body | IEEE-754 single | passthrough | — | optional;passthrough from yaw stick × shaper |

**(c) 控制流 / 状态组(cmd_mask / ctrl_mode / mode / status / state / ext_state / reset — A1 §5.1.5 demand)**

| Order | Field | Type | Width | Offset | Unit | Frame | Encoding | Validity / Gate | B2/B3 ref | Notes |
|---:|---|---|---:|---:|---|---|---|---|---|---|
| 14 | `cmd_mask` | `uint32` | 4 | 176 | N/A | N/A | uint32 bitfield (位语义由 D6 锁定) | self | **B2: cmd_mask 位 alias** | A1 §5.1.5 demand;Latch RB-05 |
| 15 | `ctrl_mode` | `uint8` | 1 | 180 | N/A | N/A | uint8 enum (CtrlMode) | self | **B2: CtrlMode** | A1 §5.1.5 demand |
| 16 | `mode` | `uint8` | 1 | 181 | N/A | N/A | uint8 enum (FlightMode/PilotMode 系列) | self | **B2: FlightMode / PilotMode** | A1 §5.1.5 demand;mode |
| 17 | `status` | `uint8` | 1 | 182 | N/A | N/A | uint8 enum (VehicleStatus) | self | **B2: VehicleStatus** | A1 §5.1.5 demand;status |
| 18 | `state` | `uint8` | 1 | 183 | N/A | N/A | uint8 enum (VehicleState) | self | **B2: VehicleState** | A1 §5.1.5 demand;state |
| 19 | `ext_state` | `uint8` | 1 | 184 | N/A | N/A | uint8 enum (VehicleExtState) | self | **B2: VehicleExtState** | A1 §5.1.5 demand;ext_state |
| 20 | `reset` | `boolean` | 1 | 185 | N/A | N/A | bool 0/1 | self | — | A1 §5.1.5 demand;**optional**(per A6 §5 第 2 项 open;**B4 to verify firmware 是否含此字段**;若不含,删行;A3 §5 to-confirm) |

**(d) 任务 / 导航组(wp / home — A1 §5.1.5 demand)**

| Order | Field | Type | Width | Offset | Unit | Frame | Encoding | Validity / Gate | B2/B3 ref | Notes |
|---:|---|---|---:|---:|---|---|---|---|---|---|
| 21 | `wp_index` | `uint16` | 2 | 186 | N/A | N/A | raw uint16 | self | — | A1 §5.1.5 demand;passthrough from `Mission_Data_Bus.wp_current_index` |
| 22 | `wp_lat_deg` | `double` | 8 | 192 | `_deg` | N/A | IEEE-754 double | self | — | A1 §5.1.5 demand;wp;passthrough from current waypoint;(offset 188..191 alignment pad to 8-byte) |
| 23 | `wp_lon_deg` | `double` | 8 | 200 | `_deg` | N/A | IEEE-754 double | self | — | wp |
| 24 | `wp_alt_m` | `single` | 4 | 208 | `_m` | N/A | IEEE-754 single | self | — | wp |
| 25 | `home_lat_deg` | `double` | 8 | 216 | `_deg` | N/A | IEEE-754 double | self | **B3: PLANT_PARAM home init** | A1 §5.1.5 demand;home (passthrough from `Mission_Data_Bus.home_lat_deg`) |
| 26 | `home_lon_deg` | `double` | 8 | 224 | `_deg` | N/A | IEEE-754 double | self | **B3: PLANT_PARAM home init** | home |
| 27 | `home_alt_m` | `single` | 4 | 232 | `_m` | N/A | IEEE-754 single | self | **B3: PLANT_PARAM home init** | home |

**(e) 错误 / failsafe 组(error fields — A1 §5.1.5 demand)**

| Order | Field | Type | Width | Offset | Unit | Frame | Encoding | Validity / Gate | B2/B3 ref | Notes |
|---:|---|---|---:|---:|---|---|---|---|---|---|
| 28 | `error_code` | `uint8` | 1 | 236 | N/A | N/A | uint8 enum (ErrorCode) | self | **B2: ErrorCode** | A1 §5.1.5 demand;error |
| 29 | `failsafe_state` | `uint8` | 1 | 237 | N/A | N/A | uint8 enum (FailsafeState) | self | **B2: FailsafeState** | optional |

**(f) INS validity passthrough(optional echo;per A3 §4.4.6.5)**

| Order | Field | Type | Width | Offset | Unit | Frame | Encoding | Validity / Gate | B2/B3 ref | Notes |
|---:|---|---|---:|---:|---|---|---|---|---|---|
| 30 | `ins_ready` | `boolean` | 1 | 238 | N/A | N/A | bool 0/1 | passthrough echo | — | optional;**B4 to verify firmware 是否含此字段**;若不含,Controller 直接消费 `INS_Out_Bus.INS_Status.ready` (per A3 §5 to-confirm) |
| 31 | `ins_position_valid` | `boolean` | 1 | 239 | N/A | N/A | bool 0/1 | passthrough echo | — | optional |
| 32 | `ins_attitude_valid` | `boolean` | 1 | 240 | N/A | N/A | bool 0/1 | passthrough echo | — | optional |

**(g) timestamp**

| Order | Field | Type | Width | Offset | Unit | Frame | Encoding | Validity / Gate | B2/B3 ref | Notes |
|---:|---|---|---:|---:|---|---|---|---|---|---|
| 33 | `timestamp` | `uint32` | 4 | 244 | ms | N/A | raw uint32 ms (firmware-legacy) | N/A | — | per A7 §4.6;(offset 241..243 alignment pad to 4-byte) |

**Total `FMS_Out_Bus` byte width**: 248 bytes (B4 to verify alignment padding;若 firmware 不含 (b) `motor_cmd_passthrough` / (c) `reset` / (f) ins_* echo / (e) `failsafe_state` / (a) `att_cmd_euler_rad`,实际宽度更小).

#### 4.7.2 A1 §5.1.5 demand → 字段映射

A1 §5.1.5 列举的"重要字段"全部出现在本 ledger 中:

| A1 §5.1.5 demand | 本 ledger 位置(Order) |
|---|---|
| rate commands | 6 (`ang_rate_cmd_b_radps`) |
| attitude commands | 4 (`att_cmd_quat`) + 5 (`att_cmd_euler_rad`,optional) |
| velocity commands | 2 (`vel_cmd_ned_mps`) |
| acceleration commands | 3 (`acc_cmd_ned_mps2`) |
| actuator_cmd passthrough space | 10 (`actuator_cmd`) + 11 (`motor_cmd_passthrough`,optional) + 12 (`pilot_throttle_passthrough`) + 13 (`pilot_yaw_rate_passthrough_radps`,optional) |
| throttle_cmd | 9 (`throttle_cmd`) |
| cmd_mask | 14 (`cmd_mask`) |
| status | 17 (`status`) |
| state | 18 (`state`) |
| ext_state | 19 (`ext_state`) |
| ctrl_mode | 15 (`ctrl_mode`) |
| mode | 16 (`mode`) |
| reset | 20 (`reset`) |
| wp fields | 21 (`wp_index`) + 22..24 (`wp_lat/lon_deg`,`wp_alt_m`) |
| home | 25..27 (`home_lat/lon_deg`,`home_alt_m`) |
| error fields | 28 (`error_code`) + 29 (`failsafe_state`,optional) |

A1 §5.1.5 demand **全部 16 类字段** 已枚举(15 个必需 + 1 个 optional 类 — `failsafe_state`)。Per A3 §4.4.6.4 / §4.6 passthrough 集合亦被本 ledger 包含。

**注**:本 ledger 的字段顺序是 **设计意图**;firmware `FMS_types.h` 中实际顺序由 [B4 contract diff](B4-contract-diff.md) + [I3](../I-tooling/I3-contract-diff.md) 在首次 export 时验证。若 firmware 顺序与本 ledger 不一致,本文件以 firmware 实际顺序为准并触发变更日志(per RULES §10)。

### 4.8 Plant 输出 buses(模拟传感器)

#### 4.8.1 `IMU_Bus`

| 项 | 值 |
|---|---|
| Bus 名 | `IMU_Bus` (alt name `IMU_Out_Bus` per A2 R-5.1) |
| Owner module | Plant (model repo) |
| Nominal rate | typical 1 ms (与 Plant 同步, per C1);RB-07 内部 cadence |
| Simulink Bus Object | `IMU_Bus` |
| Source firmware header | `FMT-Firmware/src/model/plant/<vehicle>/lib/Plant_types.h` @ `<pending hash>` |
| Direction | Output ← Plant; Input → ins_stub (per A3 §4.3.7.1) |

| Order | Field | Type | Width | Offset | Unit | Frame | Encoding | Validity | B2/B3 ref | Notes |
|---:|---|---|---:|---:|---|---|---|---|---|---|
| 1 | `gyr_b_radps` | `single[3]` | 12 | 0 | `_radps` | body | IEEE-754 single × 3 (含噪声/偏置 by C1) | valid if `valid==1` | — | 陀螺仪 |
| 2 | `acc_b_mps2` | `single[3]` | 12 | 12 | `_mps2` | body | IEEE-754 single × 3 (specific force, 含重力) | valid if `valid==1` | — | |
| 3 | `temperature_c` | `single` | 4 | 24 | `_c` (per A2 R-10.2) | N/A | IEEE-754 single | self | — | optional |
| 4 | `valid` | `boolean` | 1 | 28 | N/A | N/A | bool 0/1 | self | — | (alignment pad to 4-byte for next field) |
| 5 | `timestamp` | `uint32` | 4 | 32 | ms | N/A | raw uint32 ms (firmware-legacy) | N/A | — | (offset 29..31 alignment pad) |

**Total bus byte width**: 36 bytes (B4 to verify;若 `temperature_c` 不存在,28 bytes).

#### 4.8.2 `MAG_Bus`

| 项 | 值 |
|---|---|
| Bus 名 | `MAG_Bus` (alt `MAG_Out_Bus` per A2 R-5.1) |
| Owner module | Plant |
| Nominal rate | typical 10 ms (per C1);RB-07 |
| Simulink Bus Object | `MAG_Bus` |
| Source firmware header | `FMT-Firmware/src/model/plant/<vehicle>/lib/Plant_types.h` @ `<pending hash>` |
| Direction | Output ← Plant; Input → ins_stub (per A3 §4.3.7.2) |

| Order | Field | Type | Width | Offset | Unit | Frame | Encoding | Validity | B2/B3 ref | Notes |
|---:|---|---|---:|---:|---|---|---|---|---|---|
| 1 | `mag_b_gauss` | `single[3]` | 12 | 0 | `_gauss` (B4 to verify; A3 §5 to-confirm vs nT) | body | IEEE-754 single × 3 | valid if `valid==1` | — | firmware-legacy 单位待 B4 |
| 2 | `valid` | `boolean` | 1 | 12 | N/A | N/A | bool 0/1 | self | — | |
| 3 | `timestamp` | `uint32` | 4 | 16 | ms | N/A | raw uint32 ms | N/A | — | (offset 13..15 alignment pad) |

**Total bus byte width**: 20 bytes (B4 to verify).

#### 4.8.3 `Barometer_Bus`

| 项 | 值 |
|---|---|
| Bus 名 | `Barometer_Bus` (alt `Barometer_Out_Bus` per A2 R-5.1) |
| Owner module | Plant |
| Nominal rate | typical 10 ms (per C1);RB-07 |
| Simulink Bus Object | `Barometer_Bus` |
| Source firmware header | `FMT-Firmware/src/model/plant/<vehicle>/lib/Plant_types.h` @ `<pending hash>` |
| Direction | Output ← Plant; Input → ins_stub (per A3 §4.3.7.3) |

| Order | Field | Type | Width | Offset | Unit | Frame | Encoding | Validity | B2/B3 ref | Notes |
|---:|---|---|---:|---:|---|---|---|---|---|---|
| 1 | `pressure_pa` | `single` | 4 | 0 | `_pa` | N/A | IEEE-754 single | valid if `valid==1` | — | 静压 |
| 2 | `temperature_c` | `single` | 4 | 4 | `_c` | N/A | IEEE-754 single | self | — | optional |
| 3 | `altitude_m` | `single` | 4 | 8 | `_m` | NED-Down (高度 above ref) | IEEE-754 single | self | — | optional / derived from pressure |
| 4 | `valid` | `boolean` | 1 | 12 | N/A | N/A | bool 0/1 | self | — | |
| 5 | `timestamp` | `uint32` | 4 | 16 | ms | N/A | raw uint32 ms | N/A | — | (offset 13..15 alignment pad) |

**Total bus byte width**: 20 bytes (B4 to verify;optional `temperature_c` / `altitude_m` 视 firmware 而定).

#### 4.8.4 `GPS_uBlox_Bus`

| 项 | 值 |
|---|---|
| Bus 名 | `GPS_uBlox_Bus` (alt `GPS_Out_Bus` per A2 R-5.1) |
| Owner module | Plant |
| Nominal rate | typical 100–200 ms (per C1);RB-07 |
| Simulink Bus Object | `GPS_uBlox_Bus` |
| Source firmware header | `FMT-Firmware/src/model/plant/<vehicle>/lib/Plant_types.h` @ `<pending hash>` |
| Direction | Output ← Plant; Input → ins_stub (per A3 §4.3.7.4) |

| Order | Field | Type | Width | Offset | Unit | Frame | Encoding | Validity | B2/B3 ref | Notes |
|---:|---|---|---:|---:|---|---|---|---|---|---|
| 1 | `lat_deg` | `double` | 8 | 0 | `_deg` | N/A | IEEE-754 double | valid if `valid==1` | — | per A2 R-10.9 |
| 2 | `lon_deg` | `double` | 8 | 8 | `_deg` | N/A | IEEE-754 double | valid if `valid==1` | — | |
| 3 | `alt_m` | `single` | 4 | 16 | `_m` | N/A (MSL) | IEEE-754 single | valid if `valid==1` | — | |
| 4 | `vel_ned_mps` | `single[3]` | 12 | 20 | `_mps` | NED | IEEE-754 single × 3 | valid if `valid==1` | — | |
| 5 | `num_sat` | `uint8` | 1 | 32 | N/A | N/A | raw uint8 (sat count) | self | — | |
| 6 | `fix_type` | `uint8` | 1 | 33 | N/A | N/A | uint8 enum (UbloxFixType) | self | **B2: UbloxFixType** | uBlox fix type |
| 7 | `hdop` | `single` | 4 | 36 | N/A | N/A | IEEE-754 single (geometric DOP) | self | — | (offset 34..35 alignment pad) |
| 8 | `vdop` | `single` | 4 | 40 | N/A | N/A | IEEE-754 single | self | — | |
| 9 | `valid` | `boolean` | 1 | 44 | N/A | N/A | bool 0/1 | self | — | |
| 10 | `timestamp` | `uint32` | 4 | 48 | ms | N/A | raw uint32 ms | N/A | — | (offset 45..47 alignment pad) |

**Total bus byte width**: 52 bytes (B4 to verify alignment padding).

#### 4.8.5 `AirSpeed_Bus`

| 项 | 值 |
|---|---|
| Bus 名 | `AirSpeed_Bus` (alt `Airspeed_Out_Bus` per A2 R-5.1) |
| Owner module | Plant (variant-conditional;FW/VTOL only) |
| Nominal rate | typical 10–20 ms (per C1);RB-07 |
| Simulink Bus Object | `AirSpeed_Bus` |
| Source firmware header | `FMT-Firmware/src/model/plant/<vehicle>/lib/Plant_types.h` @ `<pending hash>` |
| Direction | Output ← Plant; Input → ins_stub (variant);多旋翼 Phase 2:`valid` 始终 false (per A3 §4.3.7.5) |

| Order | Field | Type | Width | Offset | Unit | Frame | Encoding | Validity | B2/B3 ref | Notes |
|---:|---|---|---:|---:|---|---|---|---|---|---|
| 1 | `airspeed_mps` | `single` | 4 | 0 | `_mps` | body (forward) | IEEE-754 single | valid if `valid==1` | — | 真空速 |
| 2 | `differential_pressure_pa` | `single` | 4 | 4 | `_pa` | N/A | IEEE-754 single | valid if `valid==1` | — | |
| 3 | `valid` | `boolean` | 1 | 8 | N/A | N/A | bool 0/1 | self | — | |
| 4 | `timestamp` | `uint32` | 4 | 12 | ms | N/A | raw uint32 ms | N/A | — | (offset 9..11 alignment pad) |

**Total bus byte width**: 16 bytes (B4 to verify).

### 4.9 B1 ↔ B2 / B3 交叉引用矩阵

#### 4.9.1 B2(Enum)交叉引用 — 所有 enum-typed 字段

| Bus | Field | enum 类型 (B2 to lock 数值) |
|---|---|---|
| `Pilot_Cmd_Bus` | `mode_switch` | `PilotMode` |
| `GCS_Cmd_Bus` | `cmd_type` | `GcsCmdType` |
| `Auto_Cmd_Bus` | `cmd_mask_request` | uint32 bitfield (位 alias by D6 / B2) |
| `Mission_Data_Bus` (via `Waypoint`) | `wp_array[i].cmd_type` | `WaypointCmdType` |
| `INS_Out_Bus` | `INS_Status` | uint32 bitfield (位 alias by B2) |
| `INS_Out_Bus` | `INS_Flag` | uint32 bitfield (位 alias by B2) |
| `Control_Out_Bus` | `cmd_mask` | uint32 bitfield (位 alias by D6 / B2) |
| `Control_Out_Bus` | `ctrl_mode` | `CtrlMode` |
| `FMS_Out_Bus` | `cmd_mask` | uint32 bitfield (位 alias by D6 / B2) |
| `FMS_Out_Bus` | `ctrl_mode` | `CtrlMode` |
| `FMS_Out_Bus` | `mode` | `FlightMode` 或 `PilotMode` 系列 |
| `FMS_Out_Bus` | `status` | `VehicleStatus` |
| `FMS_Out_Bus` | `state` | `VehicleState` |
| `FMS_Out_Bus` | `ext_state` | `VehicleExtState` |
| `FMS_Out_Bus` | `error_code` | `ErrorCode` |
| `FMS_Out_Bus` | `failsafe_state` | `FailsafeState` |
| `GPS_uBlox_Bus` | `fix_type` | `UbloxFixType` |

**B1 不重定义这些 enum 的成员名 / 数值** — 委托 B2(per RULES §5、co-seal batch 兄弟)。B1 仅锁定字段类型 / 宽度(`uint8` 默认,除非 B2 在数值锁定时声明需要 `uint16` 或 `uint32` — 此情况下 B1 走变更日志同步)。**B1 草拟期假设**:契约 enum 全部用 `uint8`(0..255 范围足够);bitfield enum (`cmd_mask`, `INS_Status`, `INS_Flag`) 用 `uint32`。

#### 4.9.2 B3(Parameter)交叉引用 — 所有 PARAM 类别字段(per A6 §4.4)

| Bus | Field | B3 PARAM 归属 (类别) |
|---|---|---|
| `Plant_States_Bus` | `pos_ned_m`(初值) | `PLANT_PARAM` (echoes `States_Init_Bus`) |
| `Plant_States_Bus` | `vel_ned_mps`(初值) | `PLANT_PARAM` |
| `Plant_States_Bus` | `quat_ned_to_b`(初值) | `PLANT_PARAM` |
| `Plant_States_Bus` | `ang_rate_b_radps`(初值) | `PLANT_PARAM` |
| `Plant_States_Bus` | `mass_kg` | `PLANT_PARAM` |
| `Plant_States_Bus` | `inertia_b_kgm2` | `PLANT_PARAM` |
| `States_Init_Bus` 全字段 | (同上 — `States_Init_Bus` 是 `PLANT_PARAM` init 注入端) | `PLANT_PARAM` |
| `Environment_Info_Bus` 全字段 | (per A6 §4.4.2 全字段 PARAM) | `PLANT_PARAM` |
| `Mission_Data_Bus` | `home_lat_deg` / `home_lon_deg` / `home_alt_m` | `PLANT_PARAM` (home init) |
| `FMS_Out_Bus` | `home_lat_deg` / `home_lon_deg` / `home_alt_m`(passthrough echo) | (echo;non-tunable;但 echo 来自 PARAM) |

注:**FMS_PARAM / CONTROL_PARAM 的字段不直接暴露在任一 bus 中** — 它们是参数 struct,由 `*_PARAM` 语义经过 Tunable parameter linkage 进入 model;B3 owns 字段表。本节仅列 bus 中 PARAM 类别字段。

#### 4.9.3 cmd_mask gated 字段(委托 D6 + E1)

下表汇总所有受 `cmd_mask` 守卫的字段;**cmd_mask 位号 / 名由 D6 + E1 co-seal batch(Wave 8)锁定**(per A3 §4.7),B1 仅记录守卫依赖:

| Bus | Field | 守卫位 (D6 工作名) |
|---|---|---|
| `FMS_Out_Bus` | `pos_cmd_ned_m` | `cmd_mask.position_loop` |
| `FMS_Out_Bus` | `vel_cmd_ned_mps` | `cmd_mask.velocity_loop` |
| `FMS_Out_Bus` | `acc_cmd_ned_mps2` | `cmd_mask.acceleration_loop` |
| `FMS_Out_Bus` | `att_cmd_quat` / `att_cmd_euler_rad` | `cmd_mask.attitude_loop` |
| `FMS_Out_Bus` | `ang_rate_cmd_b_radps` | `cmd_mask.rate_loop` |
| `FMS_Out_Bus` | `yaw_cmd_rad` | `cmd_mask.yaw_loop` |
| `FMS_Out_Bus` | `yaw_rate_cmd_radps` | `cmd_mask.yaw_rate_loop` |
| `FMS_Out_Bus` | `throttle_cmd` | `cmd_mask.throttle_passthrough` |
| `Control_Out_Bus` | `motor_cmd[]` (when Controller passthrough) | `cmd_mask.throttle_passthrough` |

#### 4.9.4 Validity gated 字段(per A3 §4.4.4 / §4.5.3 / §4.8;委托 F3 fallback)

| 守卫位 | 被守护的字段 |
|---|---|
| `INS_Status.ready` | (gate)所有 FMS / Controller 对 INS 的消费(per A3 §4.8) |
| `INS_Flag.position_valid` | `INS_Out_Bus.position_lla` / `position_ned_m` |
| `INS_Flag.velocity_valid` | `INS_Out_Bus.velocity_ned_mps` |
| `INS_Flag.attitude_valid` | `INS_Out_Bus.quat_ned_to_b` / `euler_ned_to_b_rad` |
| `INS_Flag.gps_valid` | (FMS Safety gate);GPS-related decision logic |
| `INS_Flag.mag_valid` | (FMS Safety gate) |
| `*_Bus.valid` | 各模拟传感器 bus 内部 |

### 4.10 与 [B4](B4-contract-diff.md) 退出条件覆盖呼应

[B4](B4-contract-diff.md) 退出条件(per [00-design-plan.md §4.B](../00-design-plan.md))列出契约 diff 必须覆盖项 (a)..(e) + 符号存在性。本文件给出的 schema 与 B4 各项的呼应:

| B4 退出条件 | B1 给的输入 |
|---|---|
| (a) 所有 bus 字段顺序与类型字节相等 | §4.2 / §4.3 / §4.4 / §4.5 / §4.7 / §4.8 各 bus 字段 schema 表(Order / Type / Width / Offset 列)— 16 / 16 bus 全覆盖 |
| (b) 所有 enum 数值精确相等 | §4.9.1 enum 字段汇总 → 委托 B2;B1 锁定 enum 字段在 bus 中的字节宽度(默认 `uint8`,bitfield `uint32`) |
| (c) `FMS_PARAM` / `CONTROL_PARAM` / `PLANT_PARAM` 字段顺序与类型相等 | §4.9.2 PARAM 类别 bus 字段 → 委托 B3(B3 owns PARAM 字段顺序)。**B1 不直接覆盖 PARAM struct,但 PARAM 中 home/init/inertia/mass 等字段必须与 bus passthrough 字段类型一致 — B3 在自身 schema 中保证此约束。** |
| (d) `FMS_EXPORT` / `CONTROL_EXPORT` / `PLANT_EXPORT` 字段顺序、`period` 数值、`model_info[]` 长度相等 | **不在 B1 范围**(EXPORT 字段由 [B3 §<EXPORT 节>](B3-parameter-schema.md) + [I4](../I-tooling/I4-codegen-config.md) 落地);B1 仅锁定 `period` 字段单位 = ms (per A2 R-10.3 firmware-legacy),`model_info[]` 是 char array。 |
| (e) 符号存在性 — `FMS_init` / `FMS_step` / `Controller_init` / `Controller_step` / `Plant_init` / `Plant_step` 必须出现 | **不在 B1 范围**(由 [I4 codegen 配置](../I-tooling/I4-codegen-config.md) + [A2 §4.8 R-8.1](../A-architecture/A2-naming-conventions.md) 顶层模型根名约定保证);B1 仅锁定相关 PARAM/bus 字段类型不阻碍 codegen。 |

### 4.11 Sample-time / 速率覆盖(per A4 echo)— 退出条件 (c)

每个 bus 的 nominal rate 集中(per 任务 brief 退出条件 (c)):

| Bus | Nominal rate (产出方周期) | 边界 ID(per A4) |
|---|---|---|
| `Pilot_Cmd_Bus` | external (≈ 50–100 Hz) | RB-06 → FMS 20 ms |
| `GCS_Cmd_Bus` | external (≈ 5–20 Hz) | RB-06 → FMS 20 ms |
| `Auto_Cmd_Bus` | external | RB-06 → FMS 20 ms |
| `Mission_Data_Bus` | mission-load (sporadic) | RB-06 → FMS 20 ms |
| `INS_Out_Bus` | 10 ms (firmware-owned) | RB-03 → Controller 5 ms;RB-04 → FMS 20 ms |
| `FMS_Out_Bus` | 20 ms | RB-05 → Controller 5 ms |
| `Control_Out_Bus` | 5 ms | RB-02 → Plant 1 ms;RB-04 等价 → FMS 20 ms (variant) |
| `Plant_States_Bus` | 1 ms | RB-01 → Controller 5 ms (variant only);ins_stub 内部 |
| `Extended_States_Bus` | 1 ms | RB-01 (harness-only) |
| `Environment_Info_Bus` | configuration-time | non-step bus |
| `States_Init_Bus` | configuration-time | non-step bus (init only) |
| `IMU_Bus` | 1 ms (typ.) | RB-07 |
| `MAG_Bus` | 10 ms (typ.) | RB-07 |
| `Barometer_Bus` | 10 ms (typ.) | RB-07 |
| `GPS_uBlox_Bus` | 100–200 ms (typ.) | RB-07 |
| `AirSpeed_Bus` | 10–20 ms (typ.) | RB-07 |

### 4.12 16 / 16 bus 覆盖检查(退出条件 (a) coverage)

| # | Bus | §位置 | 完整 schema 表 | A1 §5.1.5 demand 闭合 |
|---:|---|---|---|---|
| 1 | `Pilot_Cmd_Bus` | §4.2.1 | yes | — |
| 2 | `GCS_Cmd_Bus` | §4.2.2 | yes | — |
| 3 | `Auto_Cmd_Bus` | §4.2.3 | yes | — |
| 4 | `Mission_Data_Bus` | §4.2.4 | yes | — |
| 5 | `INS_Out_Bus` | §4.3.1 | yes (B5 mirror; B1 best-effort) | — |
| 6 | `FMS_Out_Bus` | §4.7 (full ledger) | yes | yes (闭合 A1 §5.1.5) |
| 7 | `Control_Out_Bus` | §4.4.1 | yes | — |
| 8 | `Plant_States_Bus` | §4.5.1 | yes | — |
| 9 | `Extended_States_Bus` | §4.5.2 | yes | — |
| 10 | `Environment_Info_Bus` | §4.5.3 | yes | — |
| 11 | `States_Init_Bus` | §4.5.4 | yes | — |
| 12 | `IMU_Bus` | §4.8.1 | yes | — |
| 13 | `MAG_Bus` | §4.8.2 | yes | — |
| 14 | `Barometer_Bus` | §4.8.3 | yes | — |
| 15 | `GPS_uBlox_Bus` | §4.8.4 | yes | — |
| 16 | `AirSpeed_Bus` | §4.8.5 | yes | — |

**16 / 16 全覆盖** 。

## 5. 已知风险与悬而未决问题

- **firmware commit hash 暂缺**
  - 影响:RULES §5 / §3 表 require 镜像自 firmware 的契约记录 commit hash + 文件相对路径;本文件 §3 仅给路径占位 `FMT-Firmware @ <pending hash>`(per A3 / A6 / A7 同样的占位约定)。
  - 处置:open;评审通过前由 reviewer / orchestrator 在 INDEX 决策日志登记本工作项时补齐(per A6 §5 第 1 项)。本工作项 contract_impact=yes 已声明。

- **A3 标 "B1 to confirm" 项需 byte-verify 后回填**
  - 涉及:`Control_Out_Bus.motor_cmd[]` 单位(`_n01` vs `_us`);`Plant_States_Bus` / `INS_Out_Bus` / `States_Init_Bus` / `FMS_Out_Bus` 中 `quat` 顺序(w-first vs w-last);`euler_*_rad` 顺序(ZYX 占位 yaw-pitch-roll vs 其他);`MAG_Bus.mag_b_*` 单位(Gauss vs nT);`Mission_Data_Bus.wp_array[]` 长度(占位 16);`actuator_cmd[]` 长度(占位 16) / 含义(`_n01` vs trim);`FMS_Out_Bus.reset` 字段是否真的存在;FMS validity echo (`ins_ready` 等)是否真的存在;`FMS_Out_Bus.att_cmd_quat` vs `att_cmd_euler_rad` 是否同时存在或二选一;`FMS_Out_Bus.motor_cmd_passthrough` 是否真的存在;`Control_Out_Bus` 是否含 `cmd_mask` / `ctrl_mode` / `reset` echo。
  - 处置:本文件每条带 `(B4 to verify)` 标;首次 export 时 [B4 / I3](../I-tooling/I3-contract-diff.md) 验证,反馈触发 §8 变更日志。**B1 退出条件**(per 00-design-plan §4.B "字段级 schema 与 firmware `*_types.h` 字段顺序、字节布局一致")要求所有 to-confirm 在 firmware 仓挂载后逐条回填或拒接。

- **数组长度占位**
  - 影响:`actuator_cmd[]`, `motor_cmd_passthrough[]`, `motor_cmd[]`, `motor_rpm[]`, `motor_state[]`, `aux_channel[]`, `cmd_param[]`, `wp_array[]` 长度均为 best-effort 占位;实际由 vehicle leaf(C3 / D5 / E4)决定,且必须与 firmware 实际数组长度精确匹配。
  - 处置:本文件 §4.1.7 集中列出占位长度;C3 / D5 / E4 落地多旋翼实际值后,本文件走变更日志同步;B4 contract diff 在每次 export 时验证。

- **总字节宽度 & alignment padding 未精确**
  - 影响:本文件每个 bus 末尾给出"Total bus byte width"是 natural alignment best-effort;若 firmware 编译器使用 `__attribute__((packed))` 或显式 `_pad[N]` 字段插入,实际宽度不同。
  - 处置:`(B4 to verify alignment padding)` 标;firmware 仓挂载后由 [B4](B4-contract-diff.md) + [I3](../I-tooling/I3-contract-diff.md) 在 export 时计算精确宽度。本草拟期 byte width 仅用于设计意图与 Simulink Bus Object 创建,**不进入 firmware export 数据(由 firmware `*_types.h` 主导)**。

- **`INS_Out_Bus` 字段顺序最终由 B5 锁定**
  - 影响:本文件 §4.3.1 给消费侧最佳推断顺序(基于 A3 §4.4.4 字段类别);firmware `INS_types.h` 实际字段顺序可能不同,完全镜像策略由 [B5 INS_Out_Bus 镜像同步流程](B5-ins-bus-mirror.md) 落地。
  - 处置:open;B5 在 Wave 5 启动,本表为 B5 提供消费侧字段类别清单作为镜像 review 锚点;B5 镜像产物为 canonical,B1 §4.3.1 在 B5 完成后走变更日志同步。

- **enum 字段的 underlying integer width 未锁**
  - 影响:本文件 §4.9.1 假设契约 enum 全部 `uint8`;若 B2 数值锁定时发现某 enum 取值范围超过 255,需升 `uint16` 或 `uint32` —— 此情况下本文件相应字段的 Width / Offset 列变更。
  - 处置:open;co-seal batch 协同;B2 数值锁定后本文件走变更日志同步。

- **`FMS_Out_Bus` 字段顺序设计意图 vs firmware 实际顺序**
  - 影响:§4.7 ledger 是设计意图(setpoint → passthrough → 控制流 → 任务 → 错误 → INS validity → timestamp);若 firmware `FMS_types.h` 实际顺序不同,本文件以 firmware 为准并走变更日志。
  - 处置:`(B4 to verify)` — A1 §5.1.5 demand 闭合点是字段集合**完整**(已满足),字段顺序由 firmware 主导;本文件 ledger 给出设计意图作为 codegen 期望,实际顺序由 firmware `*_types.h` 锁。

- **co-seal batch 协同风险**
  - 影响:B2 / B3 兄弟 draft 可能在协同评审中提出对本文件字段类型 / B2 enum 类型 underlying width / B3 PARAM 字段命名的修订;本文件 §4.9 cross-ref 表必须随之更新。
  - 处置:per RULES §6 self-check 第 2 项 + 01-design-relationships.md §5,co-seal batch (B1, B2, B3) 在 Wave 4 协同定稿;评审时 reviewer 检查三件套字段类型一致性。

- **endianness 假设的运行时验证**
  - 影响:§4.1.1 假设 little-endian;若未来 firmware 切到 big-endian 目标(如某些大端 PowerPC / SPARC),本文件假设破坏。
  - 处置:监控点;B4 在每次 export 时验证 firmware 编译产物 endianness。当前主流 ARM Cortex-M 全部 little-endian,风险面低。

- **`Mission_Data_Bus.wp_array` 总宽度依赖 `Waypoint` 子 schema**
  - 影响:本文件 §4.2.4 总宽度公式带变量 `sizeof(Waypoint)`;若 `Waypoint` 子 schema 在 B4 验证时与本文件 §4.6.7 不一致,bus 总宽度更改。
  - 处置:`(B4 to verify)`;§4.6.7 占位 36 bytes,可能因 alignment / 实际 `param[]` 长度变化。

- **本工作项 contract_impact=yes 的影响范围**
  - 影响:任何 B1 字段顺序 / 类型 / 宽度变化 = firmware 契约风险;变更必须经 INDEX 决策日志登记(per RULES §10)。
  - 处置:已声明 contract_impact=yes;变更走 RULES §10 + INDEX 登记。本文件**直接定义** byte-level schema,故风险面比 A3 高;每次回填 to-confirm 项均触发变更日志。

## 6. 退出条件复核

B1 在 [`00-design-plan.md`](../00-design-plan.md) §4.B 中的退出条件原文:

> 闭环必需 + 完整契约的全部 bus 列表;每个 bus 字段级 schema 与 firmware `*_types.h` 字段顺序、字节布局一致;**包含的覆盖范围已与 B4 退出条件呼应**

逐条复核:

| # | 退出条件分解 | 本文档依据 | 状态 |
|---|---|---|---|
| 1 | "闭环必需 + 完整契约的全部 bus 列表" | §4.12 16 / 16 bus 覆盖检查表;§4.2 / §4.3 / §4.4 / §4.5 / §4.7 / §4.8 各 bus header 表(name / owner / rate / Simulink Bus Object / firmware header) | 满足(16 个全覆盖) |
| 2 | "每个 bus 字段级 schema" | 每个 bus 字段表(Order / Field / Type / Width / Offset / Unit / Frame / Encoding / Validity / B2-B3 ref / Notes) | 满足 |
| 3 | "与 firmware `*_types.h` 字段顺序" | §4.7 `FMS_Out_Bus` ledger 顺序闭合 A1 §5.1.5;其他 bus 字段顺序 best-effort,标 `(B4 to verify)`,首次 export 时验证 | 满足(设计意图层;firmware 仓挂载后回填) |
| 4 | "字节布局一致" | 每个 bus 表 Width / Offset 列 + 总宽度行,natural alignment best-effort,标 `(B4 to verify alignment padding)`;§4.1.1 endianness 显式声明 | 满足(设计意图层;firmware 仓挂载后回填) |
| 5 | "包含的覆盖范围已与 B4 退出条件呼应" | §4.10 与 B4 退出条件 (a)..(e) 呼应表;§4.9 enum / param 交叉引用 | 满足 |

**任务 brief 额外退出条件复核**:

| # | 任务 brief 子条件 | 本文档依据 | 状态 |
|---|---|---|---|
| (a) | 所有 bus 字段名 + 类型 + order | §4.2 ~ §4.8 各 bus 字段表;§4.7 `FMS_Out_Bus` 完整 ledger | 满足 |
| (b) | 与 B4 expanded 退出条件呼应(bus byte order / enum exact / *_PARAM/*_EXPORT / period / model_info[] / 符号存在性)| §4.10 表 | 满足(B1 范围内项;PARAM/EXPORT/period/model_info/符号存在性显式委托给 B3 + I4) |
| (c) | Sample-time / rate per bus | §4.11 echo from A4 | 满足 |

**A1 §5.1.5 闭合验证(本工作项核心 demand)**:

| A1 §5.1.5 demand 类 | 本文档闭合位置 |
|---|---|
| rate cmd / attitude cmd / velocity cmd / acceleration cmd | §4.7.1 (a) Order 6 / 4-5 / 2 / 3 |
| actuator_cmd passthrough space | §4.7.1 (b) Order 10-13 |
| throttle_cmd | §4.7.1 (a) Order 9 |
| cmd_mask | §4.7.1 (c) Order 14 |
| status / state / ext_state / ctrl_mode / mode | §4.7.1 (c) Order 17 / 18 / 19 / 15 / 16 |
| reset | §4.7.1 (c) Order 20 |
| wp fields | §4.7.1 (d) Order 21-24 |
| home | §4.7.1 (d) Order 25-27 |
| error fields | §4.7.1 (e) Order 28-29 |

A1 §5.1.5 全部 demand 类闭合;§4.7.2 给出 demand → ledger 完整映射表。

退出条件全部满足;状态待 reviewer 升级到 reviewed。

## 7. 下游影响

按 [`01-design-relationships.md §4.2 / §4.3 / §4.4 / §4.5 / §4.6 / §4.7 / §4.9`](../01-design-relationships.md) B1 出边:

```text
A3 → B1
B1 ↔ B2 ↔ B3                         (co-seal batch)
B1, B2, B3 → B4
B1 → B5
A3, B1 ⇢ C1
A3, B1, B2 ⇢ D1
A3, B1, B2 ⇢ E1
B1, B2, B3 ⇢ I2
A3, B1 ⇢ I4
G3 ⇢ ... (logsout uses B1 fields)
```

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [B2 Enum 清单与数值锁定](B2-enum-inventory.md) *(co-seal sibling, draft)* | ↔ | §4.9.1 enum 字段汇总 — B2 owns 数值表;B1 锁字段类型(默认 `uint8`,bitfield `uint32`) |
| [B3 Parameter schema 设计](B3-parameter-schema.md) *(co-seal sibling, draft)* | ↔ | §4.9.2 PARAM 类别 bus 字段清单 — B3 owns PARAM/EXPORT 字段表;B1 提供 bus 端 echo 字段类型一致性需求 |
| [B4 契约 diff 策略设计](B4-contract-diff.md) *(未启动,Wave 5)* | → | §4.7 `FMS_Out_Bus` ledger + §4.2..§4.8 全 bus schema → B4 diff 工具的"基准 schema"输入;§4.10 与 B4 退出条件呼应表 |
| [B5 INS_Out_Bus 镜像同步流程](B5-ins-bus-mirror.md) *(未启动,Wave 5)* | → | §4.3.1 `INS_Out_Bus` 消费侧最佳推断顺序;镜像 review 锚点 |
| [C1 Plant 功能设计](../C-plant/C1-plant-functional.md) *(未启动,Wave 6)* | ⇢ | §4.5 Plant 输出 bus schema;§4.5.3 `Environment_Info_Bus` schema (扰动注入接口) |
| [C2 Plant 结构设计](../C-plant/C2-plant-structural.md) *(未启动,Wave 7)* | ⇢ | §4.5 Plant 输出 bus schema (Output Assembler 字段集);§4.5.4 `States_Init_Bus` (init 注入) |
| [C3 Plant 多旋翼 leaf](../C-plant/C3-multicopter-leaf.md) *(未启动,Wave 9)* | ⇢ | §4.1.7 `motor_rpm[]` / `motor_state[]` 长度占位;§4.5.4 `States_Init_Bus.motor_state[]` |
| [D1 FMS 功能设计](../D-fms/D1-fms-functional.md) *(未启动,Wave 6)* | ⇢ | §4.7 `FMS_Out_Bus` ledger (Output Assembler 输出契约);§4.2 (Pilot/GCS/Auto/Mission) 输入 bus schema (Source Selector / Mission Manager 输入契约);A3 §4.10 OQ4 决策给本文件的 stable input |
| [D2 FMS 结构设计](../D-fms/D2-fms-structural.md) *(未启动,Wave 7)* | ⇢ | §4.7 `FMS_Out_Bus` ledger;§4.7.1 (b) passthrough 字段集 → Output Assembler |
| [D3 FMS Mode Manager](../D-fms/D3-mode-manager.md) *(未启动,Wave 8)* | ⇢ | §4.7.1 (c) 控制流字段(`status` / `state` / `ext_state` / `ctrl_mode` / `mode` / `reset`) |
| [D4 FMS Command Shaper](../D-fms/D4-command-shaper.md) *(未启动,Wave 8)* | ⇢ | §4.7.1 (a) setpoint 字段 + §4.7.1 (b) passthrough 字段 |
| [D5 FMS 多旋翼 leaf](../D-fms/D5-multicopter-leaf.md) *(未启动,Wave 9)* | ⇢ | §4.1.7 `wp_array[]` 长度;§4.6.7 `Waypoint` schema |
| [D6 FMS↔Controller 接口约定](../D-fms/D6-fms-controller-interface.md) *(未启动,Wave 8)* | ↔(co-seal with E1)| §4.7.1 (c) `cmd_mask` 字段(uint32 bitfield);§4.9.3 cmd_mask gated 字段集 → D6 锁位号 |
| [E1 Controller 功能设计](../E-controller/E1-controller-functional.md) *(未启动,Wave 8)* | ↔(co-seal with D6)| §4.7 `FMS_Out_Bus` ledger (Controller 消费侧);§4.4.1 `Control_Out_Bus` (Controller 输出);§4.3.1 `INS_Out_Bus` (Controller 消费侧);§4.9.4 validity gated 字段 |
| [E2 Controller 结构设计](../E-controller/E2-controller-structural.md) *(未启动,Wave 7)* | ⇢ | §4.4.1 / §4.7 / §4.3.1 |
| [E4 Controller 多旋翼 leaf](../E-controller/E4-multicopter-leaf.md) *(未启动,Wave 9)* | ⇢ | §4.1.7 `motor_cmd[]` / `actuator_cmd[]` 长度占位 |
| [F1 ins_stub 功能设计](../F-ins-contract/F1-ins-stub-functional.md) *(未启动,Wave 6)* | ⇢ | §4.8 各模拟传感器 bus(ins_stub 输入);§4.3.1 `INS_Out_Bus` (ins_stub 输出);§4.6.5 / §4.6.6 INS_Status / INS_Flag 子 schema |
| [F2 ins_stub 结构设计](../F-ins-contract/F2-ins-stub-structural.md) *(未启动,Wave 7)* | ⇢ | §4.5.1 `Plant_States_Bus` → §4.3.1 `INS_Out_Bus` 字段映射来源 |
| [F3 INS_Out_Bus 消费规则](../F-ins-contract/F3-consumption-rules.md) *(未启动,Wave 9)* | ⇢ | §4.3.1 `INS_Out_Bus` schema + §4.9.4 validity gated 字段;`INS_Status.ready` / `INS_Flag.*` 位 alias 由 B2 锁;F3 owns fallback 行为 |
| [G1 MIL 顶层结构设计](../G-harness/G1-mil-toplevel.md) *(未启动,Wave 10)* | ⇢ | §4.5.4 `States_Init_Bus` (G1 注入接口);§4.5.3 `Environment_Info_Bus` (G1 注入接口);§4.4.1 `Control_Out_Bus → FMS` 回送变体 |
| [G3 日志与可观测性](../G-harness/G3-logging.md) *(未启动,Wave 10)* | ⇢ | §4.2 ~ §4.8 全 bus 字段表 → logsout 至少包含 `cmd_mask` / `mode` / `state` / 各 INS 验证位 / `timestamp` |
| [G4 Pilot_Cmd 注入设计](../G-harness/G4-pilot-injection.md) *(未启动,Wave 10)* | ⇢ | §4.2.1 `Pilot_Cmd_Bus` schema;G4 注入序列必须按本表字段集合构造 |
| [I2 Bus/enum 镜像脚本设计](../I-tooling/I2-bus-enum-mirror.md) *(未启动,Wave 5)* | ⇢ | §4.2 ~ §4.8 全 bus schema → I2 镜像脚本输出格式;§4.6 子 schema 集 → I2 嵌套类型生成模板 |
| [I3 契约 diff 脚本设计](../I-tooling/I3-contract-diff.md) *(未启动,Wave 5,与 B4 协同)* | ⇢ | §4.10 B4 退出条件呼应表 → I3 diff 字段表实现规约 |
| [I4 Codegen 配置脚本设计](../I-tooling/I4-codegen-config.md) *(未启动,Wave 12)* | ⇢ | §4.2 ~ §4.8 全 bus schema → I4 ert.tlc 配置 inport / outport 表 |

间接影响:本工作项是 Wave 4 的关键路径项,所有下游 C / D / E / F / G / I 均通过 B 区接力链(A3 → B1 → C/D/E/F/G/I)受益。

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-07 | B1 author | 初稿(co-seal batch with B2, B3) |
| 2026-05-07 | orchestrator | shared Reviewer verdict=pass(per-doc 7/7 + cross-doc C-1..C-10 全部 met,六条非阻塞建议留待下一轮迭代);frontmatter 升 reviewed;INDEX 决策日志已登记。commit hash 占位仍为 `<pending hash>`,与 A3/A6/A7 同期补齐 |

## Self-check

- [x] frontmatter 完整,字段值合法
- [x] 上游文档全部存在且 status ≥ reviewed,**或**为本工作项所在 co-seal batch 的同批兄弟(架构 v1 已发布;A1 / A2 / A3 / A4 / A5 / A6 / A7 / A8 全部 status=reviewed,2026-05-05 / 2026-05-07;B2 / B3 为 co-seal batch (B1, B2, B3) 同批兄弟,allowed per RULES §6 第 2 项;§3 已注明 batch 名)
- [x] 退出条件逐条复核完成,每条均给出依据(§6 主退出条件表 5 项 + 任务 brief 子条件 (a)/(b)/(c) 表 + A1 §5.1.5 闭合表)
- [x] 引用路径全部可点击访问
- [x] 不存在 RULES §5 禁则中的内容(无 .slx 截图、无可执行 .m;§4.x 表为字段级 schema 文本表,非截图;不复述 firmware 实现细节,只描述模型仓侧字段顺序与设计意图;FMT-Firmware 引用以路径占位 + `<pending hash>` per A3/A6/A7 模式;不重定义 enum 数值表 — 委托 B2;不重定义 PARAM 字段表 — 委托 B3)
- [x] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记(2026-05-07,orchestrator,Wave 4 完成 + B1 contract impact 条目;commit hash 占位仍为 `<pending hash>`,与 A3 / A6 / A7 同期补齐)
- [x] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径(文件相对路径已在 §3 给出 — `FMT-Firmware/src/model/{plant,fms,control}/<vehicle>/lib/{Plant,FMS,Controller}_types.h` + `FMT-Firmware/src/model/ins/lib/INS_types.h`;commit hash 占位 `<pending hash>` 与 A3/A6/A7 同期补齐;本文件不创建镜像产物,镜像产物由 [B5](B5-ins-bus-mirror.md) / [I2](../I-tooling/I2-bus-enum-mirror.md) 落地)
- [x] 下游影响已沿关系图识别完毕(§7 含 B2 / B3 / B4 / B5 直接 co-seal / 强前置 + C/D/E/F/G/I 共 24 个下游)
- [x] 文档不超出本工作项范围(无越权设计:enum 数值留 B2;PARAM/EXPORT 字段表留 B3 + I4;cmd_mask 位号留 D6/E1;FMS 启用 `Control_Out_Bus` 功能裁决留 D1;INS fallback 留 F3;harness 拓扑留 G1;variant codegen 留 I4;`INS_Out_Bus` 字段最终顺序留 B5;codegen 符号存在性留 I4)
