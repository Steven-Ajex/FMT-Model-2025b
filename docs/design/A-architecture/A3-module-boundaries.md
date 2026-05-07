---
work_item: A3
title: 模块边界信号清单细化
upstream: ["架构 v1", "A1", "A2", "A4", "A5", "A6", "A7", "A8"]
contract_impact: yes
status: reviewed
authored_at: 2026-05-07
last_reviewed_at: 2026-05-07
reviewer_verdict: pass
---

# A3 模块边界信号清单细化

## 1. 目的

对 FMT-Model-2025b 三个**模型仓侧**模块(Plant / FMS / Controller)输入与输出 bus 的**字段级清单**与方向给出一份冻结契约级界面台账,作为 [B1 Bus 清单与 schema 设计](../B-contracts/B1-bus-inventory.md) 落到字节顺序前的上游决策:每个字段标注其类型(尽力,或留 "B1 to confirm")、单位(按 [A2](A2-naming-conventions.md) §4.10)、坐标系(按 [A2](A2-naming-conventions.md) §4.10.2)、产出方周期(按 [A4](A4-rate-boundaries.md) §4.1)、reset-time 稳态值类别(按 [A6](A6-init-reset-contract.md) §4.4)、可选性 / validity 依赖 / cmd_mask 守卫 / 变体条件等语义标签。INS 由 firmware 拥有(架构 v1 §3 / §12.3),故 A3 仅枚举 `INS_Out_Bus` 的**消费侧契约**字段类别,不重述 INS 内部状态。

## 2. 范围

**在范围:**

- Plant / FMS / Controller 三个模型仓侧顶层模块的**全部输入 bus 与输出 bus 的字段级清单**
- 每个 bus 的**方向**(Input / Output / Optional / Variant-conditional)
- 每个字段的:**字段名**、**类型(best-effort 或 B1 to confirm)**、**单位**(per A2)、**坐标系**(body / NED / world / N/A,per A2)、**产出方 nominal rate**(per A4)、**reset-time 稳态值类别**(PARAM / HARDCODED / ZERO / PRESERVED,per A6)、**备注**(passthrough、validity 依赖、cmd_mask 守卫、optional、variant-conditional)
- A1 §5.1.4 闭合:FMS 输入哪些 mandatory / 哪些 optional / `Control_Out_Bus` 何时被读取
- A1 §7.2 闭合:`FMS_Out_Bus` passthrough 字段集枚举(architecture v1 §5.1.5 actuator_cmd passthrough + A1 衍生)
- A1 §8.3 / OQ4:bus 边界级记录,功能裁决留 D1
- 16 个 firmware-bus 引用全覆盖:`Pilot_Cmd_Bus` / `GCS_Cmd_Bus` / `Auto_Cmd_Bus` / `Mission_Data_Bus` / `INS_Out_Bus` / `FMS_Out_Bus` / `Control_Out_Bus` / `Plant_States_Bus` / `Extended_States_Bus` / `Environment_Info_Bus` / `States_Init_Bus` / `IMU_Bus` / `MAG_Bus` / `Barometer_Bus` / `GPS_uBlox_Bus` / `AirSpeed_Bus`

**不在范围(由其他工作项处理):**

- bus 字段的**字节顺序、padding、内存布局** — 由 [B1 Bus 清单与 schema 设计](../B-contracts/B1-bus-inventory.md) 锁定
- enum 数值与符号 — 由 [B2 Enum 清单与数值锁定](../B-contracts/B2-enum-inventory.md) 锁定
- 参数 struct 字段清单与默认值 — 由 [B3 Parameter schema 设计](../B-contracts/B3-parameter-schema.md) 落地
- INS 内部状态、`INS_Out_Bus` 来源 — firmware 拥有(架构 v1 §3 / §12.3);A3 仅在消费侧引用其字段类别
- 字段具体数值(reset 后取值的具体数字)— B3 / 各模块 leaf 设计落地
- bus 字段在 Simulink 中的实际 Signal 表达 — 由 B1 + I2(镜像脚本)落地
- 字段在 cmd_mask 之下的具体环路裁剪规则 — 由 [E1 / D6](../D-fms/D6-fms-controller-interface.md) co-seal batch 落地
- harness-only / SIH-only Variant 接入的物理拓扑 — 由 [G1 MIL 顶层结构设计](../G-harness/G1-mil-toplevel.md) / [A5 §4.5](A5-variant-strategy.md) 落地
- INS 消费侧 fallback 行为 — 由 [F3 INS_Out_Bus 消费规则](../F-ins-contract/F3-consumption-rules.md) 落地;A3 仅给出消费方依赖矩阵

## 3. 依赖

| 上游 | 引用位置 | 用途 |
|---|---|---|
| 架构 v1 §5.1.4 / §5.1.5 / §5.1.6 / §5.1.7 / §5.1.8 / §7 / §8 / §9 | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | 三模块输入/输出 bus 的契约级清单与 boundary rule;passthrough 字段提示 |
| 架构 v1 §13 模块周期表 | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | 每个字段标注产出方 nominal rate(Plant 1 ms / Controller 5 ms / FMS 20 ms / INS 10 ms firmware-owned) |
| [A1 架构 v1 评审与缺口闭合](A1-v1-review.md) (status: reviewed, 2026-05-05) | A1 §4.2 §5.1.4 / §5.1.6 / §5.1.7 / §7.2 / §8.1 / §8.3 / §8.4;A1 §4.3 OQ4 | 本工作项必须闭合的具体缺口清单 |
| [A2 命名与目录约定设计](A2-naming-conventions.md) (status: reviewed, 2026-05-05) | A2 §4.5 / §4.10 | 字段命名风格、单位 / 坐标系 / 角度后缀;**字段名引用沿 A2 R-5.3 / R-10.1 / R-10.4 / R-10.7** |
| [A4 跨速率边界设计](A4-rate-boundaries.md) (status: reviewed, 2026-05-05) | A4 §4.1 / §4.2 RB-NN / §4.7 模板 | 每字段产出方 rate 与边界 ID(RB-01..RB-07)的引用;A4 §4.7 模板由 A3 在每 bus 头给出概览引用 |
| [A5 变体策略设计](A5-variant-strategy.md) (status: reviewed, 2026-05-05) | A5 §4.2.2 / §4.5 | variant-conditional 字段标记;Controller 的 `Plant_States_Bus` 仅 harness-only |
| [A6 Init/Reset 状态契约](A6-init-reset-contract.md) (status: reviewed, 2026-05-05) | A6 §4.4 类别表 | 每字段 reset-time 稳态值类别(PARAM / HARDCODED / ZERO / PRESERVED) |
| [A7 时间与时间戳约定](A7-time-conventions.md) (status: reviewed, 2026-05-05) | A7 §4.1 / §4.4 / §4.6 | 任一 bus 中 `timestamp` 字段一律 ms / `uint32_t` / 模启动零起点 / 模 \(2^{32}\) wrap;模型仓内部不重新打戳 |
| [A8 共享库块清单与归属](A8-shared-library-roster.md) (status: reviewed, 2026-05-05) | A8 §4.2.5 | `Validity_AND` / `Stale_Detector` / `Range_Check` 原语对 validity / staleness 字段消费的共同约定 |
| firmware `*_types.h`(契约镜像参考,**byte 级由 B1 锁定**) | `FMT-Firmware/src/model/{plant,fms,control}/<vehicle>/lib/{Plant,FMS,Controller}_types.h`;`FMT-Firmware/src/model/ins/lib/INS_types.h`;**FMT-Firmware @ \<pending hash\>**(commit hash 由 INDEX 决策日志维护方在登记本工作项时补齐;本 draft 与 [A6 §3 / A7 §3 / A6 §5 第 1 项 open](A6-init-reset-contract.md) 同样的占位约定) | 字段名 / 类型 / 顺序的 firmware 端权威来源;**A3 不复制字节布局,仅引用字段语义**(per RULES §5 / §4) |

注:firmware 字段名在 firmware-legacy 命名(如 `velocity` 不带坐标系后缀)与 A2 §4.10 后缀规则之间存在张力,按 [A2 E-5.1](A2-naming-conventions.md) firmware 已固化字段名优先;在本文件字段表中,我们以 firmware 已知字段名为准列出,**无后缀的字段在备注列标 "firmware-legacy"**。

## 4. 设计内容

### 4.1 总体架构图与 bus 分层

承袭架构 v1 §7 / §8 / §9 与 [A4 §4.1](A4-rate-boundaries.md):

```text
   外部源(50/100/...Hz async)
     Pilot_Cmd_Bus, GCS_Cmd_Bus,
     Auto_Cmd_Bus, Mission_Data_Bus
                  │
                  ▼  (RB-06 single-element queue + latch)
              FMS  (20 ms)
                  │
                  ▼  (RB-05 ZOH + latch)
              FMS_Out_Bus
                  │
                  ▼
           Controller  (5 ms)
                  │   ▲
                  │   │ INS_Out_Bus (RB-03 ZOH from 10 ms)
                  ▼
            Control_Out_Bus
                  │
                  ▼  (RB-02 ZOH)
              Plant  (1 ms)
                  │
                  ├──> Plant_States_Bus         ──> ins_stub (harness; RB-07 fixed cadence)
                  ├──> Extended_States_Bus
                  ├──> Environment_Info_Bus     (note: typically Plant input; see §4.5)
                  └──> 传感器 bus(IMU/MAG/Barometer/GPS_uBlox/AirSpeed)
                                                 │
                                                 ▼
                                              ins_stub (MIL only) ─> INS_Out_Bus ─> FMS / Controller
```

**关键架构事实**(本文件用于字段方向决策):

1. Plant、FMS、Controller 是**模型仓侧**的三个 firmware-export 顶层模型(架构 v1 §7 / §8)。INS 由 firmware 拥有(架构 v1 §12.3),`INS_Out_Bus` 在模型仓侧仅作为**消费契约**;由 [A6 §4.5.3](A6-init-reset-contract.md) 协调消费侧 gate。
2. `States_Init_Bus`、`Environment_Info_Bus` 是 Plant 的**输入**(架构 v1 §5.1.7 / §8.1)。
3. `FMS_Out_Bus` 是 FMS 的唯一输出(架构 v1 §5.1.5 / §7.2 / §8.3),内部含 actuator_cmd passthrough 空间(§4.6 内详述)。
4. `Control_Out_Bus` 是 Controller 的输出,Plant 的输入,**FMS 的可选输入**(架构 v1 §5.1.4 / §8.3;OQ4 见 §4.10)。
5. 速率链:1 ms(Plant)/ 5 ms(Controller)/ 10 ms(INS, fw)/ 20 ms(FMS)整数倍同相对齐(A4 §4.5)。

### 4.2 列定义与表头约定

每个 bus 表使用以下列(在每个表头不再重复说明):

| 列 | 含义 |
|---|---|
| Field | 字段名(沿 firmware 已固化名;新增字段沿 A2 §4.5 R-5.3 + §4.10 后缀规则) |
| Type | 类型(尽力;不确定标 "B1 to confirm"。基础类型仅作语义指针,**字节布局由 B1 锁定**) |
| Unit | 单位后缀(per A2 §4.10.1)。N/A 表示无量纲(布尔、enum、计数等) |
| Frame | 坐标系(NED / body / world / vehicle / N/A,per A2 §4.10.2) |
| Rate | 产出方 nominal rate(ms;per A4 §4.1)。引用 RB-NN 的边界由 §4.x 节头给出 |
| Reset | reset-time 稳态值类别(PARAM / HARDCODED / ZERO / PRESERVED,per A6 §4.4) |
| Notes | 可选性 / validity / cmd_mask 守卫 / passthrough / variant 等 |

约定补充:

- 任一 `timestamp` 字段类型 = `uint32_t`,Unit = `ms`,Frame = N/A,wrap 行为 per A7 §4.3。本文件不在每行重述。
- 任一 enum 字段类型由 B2 锁定;A3 仅给字段名与语义。
- "B1 to confirm" 表示 A3 不能在不引入字节级假设的前提下定型,留 B1 byte-verify 后回填;为保留可追溯性,在本工作项 §5 风险中也登记一条总览 risk。
- 任一 firmware-legacy 字段名(无单位/坐标系后缀)在 Notes 列标 `firmware-legacy`;新增字段必须遵 A2 后缀规则。

### 4.3 Plant 模块边界

Plant 在模型仓侧周期 = **1 ms**(架构 v1 §13)。Plant 不存在于 firmware 飞行运行期(架构 v1 §3 / §15.4),仅 SIH/HIL 时进入 firmware 编译路径。

#### 4.3.1 Plant 输入/输出 bus 总览

| 方向 | Bus | 来源 / 去向 | 必/可选 / 变体条件 | 跨速率边界 ID |
|---|---|---|---|---|
| Input | `Control_Out_Bus` | Controller (5 ms) → Plant (1 ms) | mandatory(MIL 闭环必需) | RB-02 ZOH(per A4 §4.4 RB-02) |
| Input | `Environment_Info_Bus` | scenario / harness param-driven | mandatory | configuration-time 注入(non-step bus,详见 §4.5) |
| Input | `States_Init_Bus` | scenario / harness param-driven | mandatory(用于 init;init 后 PRESERVED 至下一 reset) | configuration-time 注入(non-step) |
| Output | `Plant_States_Bus` | Plant → `ins_stub`(harness)/ Controller(变体) | mandatory(MIL),Controller 直接消费仅 SIH/HIL variant(per A5 §4.5) | RB-01 sample-time-based downsample(仅 variant-conditional) |
| Output | `Extended_States_Bus` | Plant → harness / observer | mandatory(harness 日志);Controller 不消费 | 同 RB-01(variant-conditional) |
| Output | `IMU_Bus` | Plant → `ins_stub` | mandatory(MIL) | RB-07 internal cadence per §4.3.7;ins_stub 消费侧 ZOH 与 INS 100 Hz 节奏对齐 |
| Output | `MAG_Bus` | Plant → `ins_stub` | mandatory(MIL,部分变体可禁用 mag) | RB-07 |
| Output | `Barometer_Bus` | Plant → `ins_stub` | mandatory(MIL) | RB-07 |
| Output | `GPS_uBlox_Bus` | Plant → `ins_stub` | mandatory(MIL,GPS-denied 场景由 H4 故障注入) | RB-07 |
| Output | `AirSpeed_Bus` | Plant → `ins_stub` | optional(多旋翼默认禁用;固定翼 / VTOL 启用)— Phase 2 Plant 仍输出但 ins_stub 不消费 | RB-07 |

注:`Plant_States_Bus → Controller` 是 [A5 §4.5](A5-variant-strategy.md) harness-only Variant 接入的旁路通道,Phase 2 默认 MIL 不启用。

边界文档化模板(per A4 §4.7)按本节后续每个 bus 子节嵌入相应字段。

#### 4.3.2 Plant 输入:`Control_Out_Bus`

边界:RB-02(Controller 5 ms → Plant 1 ms,ZOH;A4 §4.4 RB-02)。

| Field | Type | Unit | Frame | Rate | Reset | Notes |
|---|---|---|---|---|---|---|
| `motor_cmd[]` | float32 array(channels TBD by E4 / leaf) | normalized 0..1 或 PWM `_us`,B1 to confirm 哪种 | N/A | 5 ms (Controller) | HARDCODED zero-thrust | 多旋翼电机推力命令;数组长度多旋翼默认 4(不在本表锁定,由 E4 / leaf) |
| `actuator_cmd[]` | float32 array | normalized 0..1 或 trim 中位 | body | 5 ms | HARDCODED trim | 舵面 / VTOL 倾转;多旋翼 Phase 2 不使用,但字段保留(passthrough 空间,per §4.6) |
| `throttle_cmd` | float32 | normalized `_n01` | N/A | 5 ms | ZERO | 总推力 / collective;具体语义由 E4 锁定 |
| `cmd_mask` | uint32 bitfield | N/A | N/A | 5 ms | HARDCODED zero(passthrough from FMS) | passthrough from `FMS_Out_Bus.cmd_mask`(per §4.6 / §4.10);Plant 不解释 cmd_mask,仅作为 trace |
| `ctrl_mode` | enum (B2 to lock) | N/A | N/A | 5 ms | HARDCODED disarmed-equiv | passthrough from FMS,Plant 不消费 |
| `reset` *(optional, per A6 §5 第 2 项 open)* | bool | N/A | N/A | 5 ms | ZERO | passthrough from FMS;Plant 不联动 reset(per A6 §4.5.2 关键约束 1) |
| `timestamp` | uint32 | ms | N/A | 5 ms | ZERO | per A7 §4.1 / §4.6;模型仓内部不篡改 |

补充:`Control_Out_Bus` 的字段集是 **Controller 输出 → Plant 输入** 的契约;字段名以 `Controller_types.h` 中 firmware 已固化为准。多旋翼 leaf(E4)落实电机数与执行器数。本表"motor_cmd / actuator_cmd / throttle_cmd"的字段名是 firmware-legacy 工作名,具体由 B1 字节验证。

#### 4.3.3 Plant 输入:`Environment_Info_Bus`

边界:configuration-time 注入(非 step-rate bus;视为常量 + 慢更新,Phase 2 默认整段仿真常量)。Reset-time 全字段 PARAM(per A6 §4.4.2 "Environment_Info_Bus 全字段 PARAM")。

| Field | Type | Unit | Frame | Rate | Reset | Notes |
|---|---|---|---|---|---|---|
| `wind_vel_ned_mps` | float32[3] | m/s | NED | configuration | PARAM | 风速向量;由 H4 故障注入扰动 |
| `air_density_kgpm3` | float32 | kg/m³ | N/A | configuration | PARAM | 大气密度;标准大气模型由 C1 决定是否常量 |
| `air_pressure_pa` | float32 | Pa | N/A | configuration | PARAM | 大气压力 |
| `air_temperature_k` | float32 | K | N/A | configuration | PARAM | 温度(per A2 R-10.1) |
| `mag_field_ned_gauss` *(or `_nt` per A2)* | float32[3] | Gauss(B1 to confirm; SI 推荐 nT 但 firmware 可能用 Gauss) | NED | configuration | PARAM | 地磁场矢量;固定大地参考 |
| `gravity_ned_mps2` | float32[3] | m/s² | NED | configuration | PARAM | 重力矢量;典型 [0, 0, 9.81] |
| `home_lat_deg` | float64 | deg(per A2 R-10.9) | N/A | configuration | PARAM | home 纬度;经纬度专项 deg |
| `home_lon_deg` | float64 | deg | N/A | configuration | PARAM | home 经度 |
| `home_alt_m` | float32 | m | N/A | configuration | PARAM | home 海拔 |

注:本 bus 在 firmware 路径下不直接由 Plant 写;模型仓 MIL harness 由 G1 注入。Phase 2 默认 H4 不启用动态扰动 ⇒ 整 bus 视为静态参数集合。

#### 4.3.4 Plant 输入:`States_Init_Bus`

边界:configuration-time 注入,仅 Plant init / reset(global TS-MIL-HARNESS)时被读取。Reset-time PARAM(per A6 §4.4.2 行 "刚体位置/速度/姿态/角速度 → PARAM(初值由 States_Init_Bus 提供)")。

| Field | Type | Unit | Frame | Rate | Reset | Notes |
|---|---|---|---|---|---|---|
| `pos_ned_m` | float32[3] | m | NED | configuration | PARAM | Plant 初始位置 |
| `vel_ned_mps` | float32[3] | m/s | NED | configuration | PARAM | Plant 初始速度 |
| `quat_ned_to_b` | float32[4] | N/A | NED→body | configuration | PARAM | Plant 初始姿态四元数(w, x, y, z 顺序由 B1 to confirm) |
| `ang_rate_b_radps` | float32[3] | rad/s | body | configuration | PARAM | Plant 初始角速度 |
| `motor_state[]` | float32 | rpm 或 normalized,B1 to confirm | N/A | configuration | ZERO 或 PARAM(per C2/C3 决定) | 电机初始状态;多旋翼默认 ZERO |

注:本 bus 字段集相对小;具体由 [C2 Plant 结构设计](../C-plant/C2-plant-structural.md) + [C3 多旋翼 leaf](../C-plant/C3-multicopter-leaf.md) 落地。"motor_state" 是工作名,B1 锁定具体字段。

#### 4.3.5 Plant 输出:`Plant_States_Bus`

边界:RB-01(MIL 默认变体 = 不存在 → Controller 不直接消费;harness-only variant 启用时为 sample-time-based downsample;A4 §4.4 RB-01)。本 bus 在 MIL 默认下被 `ins_stub` 消费(`Plant_States_Bus → ins_stub` 视为模型内通路,边界等价 RB-07 内部 cadence)。

| Field | Type | Unit | Frame | Rate | Reset | Notes |
|---|---|---|---|---|---|---|
| `pos_ned_m` | float32[3] | m | NED | 1 ms | PARAM (echoes States_Init_Bus.pos_ned_m at init) | truth state;`ins_stub` 据此构造 `INS_Out_Bus.position` |
| `vel_ned_mps` | float32[3] | m/s | NED | 1 ms | PARAM | 同上 |
| `acc_ned_mps2` | float32[3] | m/s² | NED | 1 ms | ZERO | 由 vel 微分或物理模型直出 |
| `acc_b_mps2` | float32[3] | m/s² | body | 1 ms | ZERO | body 系比力 / 加速度;`ins_stub` 用于合成 IMU 信号 |
| `quat_ned_to_b` | float32[4] | N/A | NED→body | 1 ms | PARAM | truth 姿态四元数 |
| `euler_ned_to_b_rad` | float32[3] | rad | NED→body (ZYX) | 1 ms | PARAM (derived) | (yaw, pitch, roll) 顺序由 B1 to confirm |
| `ang_rate_b_radps` | float32[3] | rad/s | body | 1 ms | PARAM | truth 角速度 |
| `ang_acc_b_radps2` | float32[3] | rad/s² | body | 1 ms | ZERO | optional;某些 plant 实现可能不输出,B1 to confirm |
| `mass_kg` | float32 | kg | N/A | 1 ms | PARAM | 质量;Phase 2 多旋翼一般常量 |
| `inertia_b_kgm2` | float32[3][3] | kg·m² | body | 1 ms | PARAM | 惯量张量(体系) |
| `motor_rpm[]` | float32 array | rpm(`_rpm` per A2 R-10.2 显式后缀,非 SI) | N/A | 1 ms | ZERO | 多旋翼电机转速;数组长度由 leaf |
| `on_ground` | bool | N/A | N/A | 1 ms | HARDCODED true | 触地标志;init 后默认在地面 |
| `timestamp` | uint32 | ms | N/A | 1 ms | ZERO | per A7 |

注:本 bus 字段名"euler_ned_to_b_rad"为模型仓侧新增建议;若 firmware-legacy 用 `euler` 无后缀,按 A2 E-5.1 沿 firmware 名称(此时 Notes 标 firmware-legacy)。具体取舍 B1 锁定。

#### 4.3.6 Plant 输出:`Extended_States_Bus`

边界:同 RB-01(harness-only)。Reset-time 全字段 ZERO 默认(per A6 §4.4.2 "Extended_States_Bus 全字段 ZERO,允许 C2 显式声明 PARAM 例外")。

| Field | Type | Unit | Frame | Rate | Reset | Notes |
|---|---|---|---|---|---|---|
| `airspeed_mps` | float32 | m/s | body (forward) | 1 ms | ZERO | 真空速;固定翼场景关键,多旋翼可为 0 |
| `aoa_rad` *(optional)* | float32 | rad | body | 1 ms | ZERO | 攻角;气动相关;多旋翼 Phase 2 不必填 |
| `aos_rad` *(optional)* | float32 | rad | body | 1 ms | ZERO | 侧滑角 |
| `dynamic_pressure_pa` | float32 | Pa | N/A | 1 ms | ZERO | 动压 |
| `flow_angle_b_rad[3]` *(optional)* | float32[3] | rad | body | 1 ms | ZERO | 飞行角扩展;variant-conditional(固定翼/VTOL) |
| `disturbance_force_b_n[3]` | float32[3] | N | body | 1 ms | ZERO | 由 `Disturbance_Injector_Shell`(A8 §4.3.1)注入的扰动力 trace |
| `disturbance_torque_b_nm[3]` | float32[3] | N·m | body | 1 ms | ZERO | 同上,扰动力矩 trace |
| `timestamp` | uint32 | ms | N/A | 1 ms | ZERO | per A7 |

注:本 bus 字段集**变体高度敏感**(架构 v1 §5.1.7 "optional others")。Phase 2 多旋翼仅消费基本几项;固定翼 / VTOL 启用 aoa / aos / flow_angle。具体集合由 [C1 Plant 功能设计](../C-plant/C1-plant-functional.md) + [C2](../C-plant/C2-plant-structural.md) 锁定。本文件给出**含 optional 标的最大集合**,B1 落字节顺序时按 firmware 已固化为准。

#### 4.3.7 Plant 输出:模拟传感器 bus(`IMU_Bus` / `MAG_Bus` / `Barometer_Bus` / `GPS_uBlox_Bus` / `AirSpeed_Bus`)

边界:RB-07(Plant 内部 1 ms 主循环 → 各传感器 cadence;A4 §4.4 RB-07)。**各传感器 cadence 由 [C1](../C-plant/C1-plant-functional.md) 锁定**(A4 不指定数值)。Reset-time 全字段 ZERO 默认。

##### 4.3.7.1 `IMU_Bus`(典型 cadence ≈ 1 ms,与 Plant 同步)

| Field | Type | Unit | Frame | Rate | Reset | Notes |
|---|---|---|---|---|---|---|
| `gyr_b_radps[3]` | float32[3] | rad/s | body | 1 ms (typ.) | ZERO | 陀螺仪;含噪声/偏置由 C1 噪声模型决定 |
| `acc_b_mps2[3]` | float32[3] | m/s² | body | 1 ms | ZERO | 比力(specific force,含重力) |
| `temperature_c` *(optional)* | float32 | °C(per A2 R-10.2 显式) | N/A | 1 ms | ZERO | IMU 温度;variant |
| `valid` | bool | N/A | N/A | 1 ms | HARDCODED false | validity 位;ins_stub gate |
| `timestamp` | uint32 | ms | N/A | 1 ms | ZERO | per A7 |

##### 4.3.7.2 `MAG_Bus`(典型 cadence ≈ 10 ms;variant-conditional)

| Field | Type | Unit | Frame | Rate | Reset | Notes |
|---|---|---|---|---|---|---|
| `mag_b_gauss[3]` | float32[3] | Gauss(B1 to confirm vs nT) | body | per C1 (≈ 10 ms) | ZERO | 磁强计读数;`firmware-legacy` 单位待 B1 |
| `valid` | bool | N/A | N/A | per C1 | HARDCODED false | |
| `timestamp` | uint32 | ms | N/A | per C1 | ZERO | |

##### 4.3.7.3 `Barometer_Bus`(典型 cadence ≈ 10 ms)

| Field | Type | Unit | Frame | Rate | Reset | Notes |
|---|---|---|---|---|---|---|
| `pressure_pa` | float32 | Pa | N/A | per C1 | ZERO | 静压 |
| `temperature_c` *(optional)* | float32 | °C | N/A | per C1 | ZERO | |
| `altitude_m` *(derived)* | float32 | m | NED-Down | per C1 | ZERO | 由 pressure 反算的气压高;某些实现仅输出 pressure |
| `valid` | bool | N/A | N/A | per C1 | HARDCODED false | |
| `timestamp` | uint32 | ms | N/A | per C1 | ZERO | |

##### 4.3.7.4 `GPS_uBlox_Bus`(典型 cadence ≈ 100–200 ms)

| Field | Type | Unit | Frame | Rate | Reset | Notes |
|---|---|---|---|---|---|---|
| `lat_deg` | float64 | deg | N/A | per C1 | ZERO | 经纬度专项 deg(per A2 R-10.9) |
| `lon_deg` | float64 | deg | N/A | per C1 | ZERO | |
| `alt_m` | float32 | m | N/A | per C1 | ZERO | MSL altitude |
| `vel_ned_mps[3]` | float32[3] | m/s | NED | per C1 | ZERO | GPS 速度 |
| `num_sat` | uint8 | N/A | N/A | per C1 | ZERO | 卫星数 |
| `fix_type` | enum (B2 to lock) | N/A | N/A | per C1 | HARDCODED no-fix-equiv | uBlox fix type;init 默认无定位 |
| `hdop` | float32 | N/A | N/A | per C1 | ZERO | 几何精度因子 |
| `vdop` | float32 | N/A | N/A | per C1 | ZERO | |
| `valid` | bool | N/A | N/A | per C1 | HARDCODED false | |
| `timestamp` | uint32 | ms | N/A | per C1 | ZERO | |

##### 4.3.7.5 `AirSpeed_Bus`(typical cadence ≈ 10–20 ms;**variant-conditional**:固定翼 / VTOL 启用,多旋翼默认禁用)

| Field | Type | Unit | Frame | Rate | Reset | Notes |
|---|---|---|---|---|---|---|
| `airspeed_mps` | float32 | m/s | body (forward) | per C1 | ZERO | 真空速 |
| `differential_pressure_pa` | float32 | Pa | N/A | per C1 | ZERO | 差压;空速管 |
| `valid` | bool | N/A | N/A | per C1 | HARDCODED false | 多旋翼 Phase 2:始终 false(传感器不存在) |
| `timestamp` | uint32 | ms | N/A | per C1 | ZERO | |

注:`AirSpeed_Bus` 是架构 v1 §5.1.7 / §8.1 列举的 "AirSpeed_Bus, optional others" 中明确字段;variant-conditional per [A5 §4.2.2](A5-variant-strategy.md) 传感器配置冻结清单。

### 4.4 FMS 模块边界

FMS 在模型仓侧周期 = **20 ms**(架构 v1 §13)。FMS 是 reset 主权威(A6 §4.3.2 / §4.5.2)。本节闭合 A1 §5.1.4 / §7.2 / §8.3 / OQ4。

#### 4.4.1 FMS 输入/输出 bus 总览(并闭合 A1 §5.1.4 mandatory/optional)

| 方向 | Bus | 来源 / 去向 | mandatory / optional / variant-conditional | 跨速率边界 | 备注 |
|---|---|---|---|---|---|
| Input | `Pilot_Cmd_Bus` | external (RC pilot stick) | **mandatory**(MIL Phase 2 闭环必需;arm/manual/altitude-hold/position-hold 全用) | RB-06 single-element queue + latch on FMS step start | source 速率不固定;按 A4 §4.4 RB-06 |
| Input | `GCS_Cmd_Bus` | external (ground station) | **optional**(若不在 H1 场景中用,可 stub-zero;但**契约级 mandatory** 即字段必须存在) | RB-06 | 来源率 5–20 Hz |
| Input | `Auto_Cmd_Bus` | external (onboard autopilot / scripts) | **optional**(Phase 2 闭环切片仅 mission-driven 子集) | RB-06 | |
| Input | `Mission_Data_Bus` | external (mission planner / loaded mission file) | **optional**(由 D1 mission 范围决定 Phase 2 是否启用) | RB-06 | |
| Input | `INS_Out_Bus` | INS (firmware-owned) / `ins_stub` (MIL) | **mandatory**(状态反馈到监管层;A6 §4.5.3 INS gate) | RB-04 sample-time-based downsample(10 ms → 20 ms,A4 §4.4 RB-04) | 消费侧 gate `ready` / `*_valid`,详见 §4.4.4 |
| Input | `Control_Out_Bus` | Controller (5 ms) → FMS (20 ms,**回送**) | **optional / transitional**(架构 v1 §17 OQ4;A1 §4.3 OQ4 "Closed by D1+A3";本工作项给 bus-边界级处置,详见 §4.10) | RB-04 等价(快→慢 sample-time-based downsample 5 ms → 20 ms);**仅当启用时存在** | A3 仅记录此输入存在与否的 bus 边界,**功能裁决是 D1**(per A1 §8.3) |
| Output | `FMS_Out_Bus` | FMS → Controller (5 ms) | **mandatory**(契约) | RB-05 ZOH + Latch(A4 §4.4 RB-05) | passthrough 字段集见 §4.6 |

闭合 A1 §5.1.4 总结:

- **Mandatory FMS 输入**:`Pilot_Cmd_Bus`、`INS_Out_Bus`(契约 / 功能皆 mandatory)
- **Optional FMS 输入**(契约级字段保留,功能/场景级可空):`GCS_Cmd_Bus`、`Auto_Cmd_Bus`、`Mission_Data_Bus`
- **Conditional FMS 输入**:`Control_Out_Bus` — 启用条件由 D1 裁决(per OQ4),A3 在 §4.10 记录 bus 边界

> 注:**契约级 mandatory** 表示该 bus 在 codegen 输出符号 / interface 层必然存在;**功能级 optional** 表示某些场景下其内部全字段可被 stub-zero,FMS 行为不变。

#### 4.4.2 FMS 输入:`Pilot_Cmd_Bus`

边界:RB-06 single-element queue + latch on FMS step start(A4 §4.4 RB-06)。

| Field | Type | Unit | Frame | Rate | Reset | Notes |
|---|---|---|---|---|---|---|
| `roll_stick_n01` | float32 | normalized -1..1 | N/A | external (≈ 50–100 Hz) | ZERO | 摇杆 roll;映射由 D4 |
| `pitch_stick_n01` | float32 | normalized -1..1 | N/A | external | ZERO | |
| `yaw_stick_n01` | float32 | normalized -1..1 | N/A | external | ZERO | |
| `throttle_stick_n01` | float32 | normalized 0..1 | N/A | external | ZERO | |
| `mode_switch` | enum (B2 to lock; PilotMode per A2 R-6.2) | N/A | N/A | external | HARDCODED disarmed-equiv | 模式开关位置 |
| `arm_switch` | bool | N/A | N/A | external | HARDCODED false | 解锁开关 |
| `kill_switch` *(optional)* | bool | N/A | N/A | external | HARDCODED false | 紧急停车;variant-conditional |
| `aux_channel[]` *(optional)* | float32 array | normalized | N/A | external | ZERO | 辅助通道;由 H1 / G4 场景脚本注入 |
| `valid` | bool | N/A | N/A | external | HARDCODED false | RC 链路有效性 |
| `timestamp` | uint32 | ms | N/A | external | ZERO | per A7;source-side timestamp 由 G4 注入 |

#### 4.4.3 FMS 输入:`GCS_Cmd_Bus` / `Auto_Cmd_Bus` / `Mission_Data_Bus`

由于这些 bus 的 firmware 字段集与多旋翼闭环切片的相关子集需 D1 锁定,A3 此处给出**已知必需字段类别 + optional 字段类别**,具体字段名留 B1 to confirm。

##### 4.4.3.1 `GCS_Cmd_Bus`(地面站发包,典型 5–20 Hz)

边界:RB-06。**Optional**(契约级 mandatory)。

| Field | Type | Unit | Frame | Rate | Reset | Notes |
|---|---|---|---|---|---|---|
| `cmd_type` | enum (B2 to lock) | N/A | N/A | external | HARDCODED idle-equiv | GCS 命令类别(arm / disarm / takeoff / land / RTL / mode change / setpoint update / ...) |
| `cmd_param[]` | float32 array (length B1 to confirm) | mixed (per cmd_type) | mixed | external | ZERO | 命令参数包;参数语义随 `cmd_type` 变 |
| `setpoint_pos_lat_deg` *(optional)* | float64 | deg | N/A | external | ZERO | 当 `cmd_type = goto_position` 时有效 |
| `setpoint_pos_lon_deg` *(optional)* | float64 | deg | N/A | external | ZERO | |
| `setpoint_pos_alt_m` *(optional)* | float32 | m | N/A | external | ZERO | |
| `setpoint_yaw_rad` *(optional)* | float32 | rad | NED→body | external | ZERO | |
| `valid` | bool | N/A | N/A | external | HARDCODED false | GCS 链路有效性 |
| `timestamp` | uint32 | ms | N/A | external | ZERO | |

##### 4.4.3.2 `Auto_Cmd_Bus`(机载自动逻辑,例如 Offboard / external scripts)

边界:RB-06。**Optional**。

| Field | Type | Unit | Frame | Rate | Reset | Notes |
|---|---|---|---|---|---|---|
| `target_pos_ned_m[3]` | float32[3] | m | NED | external | ZERO | 目标位置 |
| `target_vel_ned_mps[3]` | float32[3] | m/s | NED | external | ZERO | 目标速度 |
| `target_acc_ned_mps2[3]` *(optional)* | float32[3] | m/s² | NED | external | ZERO | 目标加速度(前馈) |
| `target_yaw_rad` | float32 | rad | NED→body | external | ZERO | 目标 yaw |
| `target_yaw_rate_radps` *(optional)* | float32 | rad/s | NED→body | external | ZERO | yaw 速率 |
| `cmd_mask_request` | uint32 bitfield | N/A | N/A | external | HARDCODED zero | Auto 模块"建议"启用的环路集;FMS 决定是否采纳(D6 / D4) |
| `valid` | bool | N/A | N/A | external | HARDCODED false | |
| `timestamp` | uint32 | ms | N/A | external | ZERO | |

##### 4.4.3.3 `Mission_Data_Bus`(任务数据,航点序列等)

边界:RB-06(载入时一次性 + 偶发更新)。**Optional**;Phase 2 闭环切片范围由 D1 / D5 决定(A1 §4.2 §16 行)。

| Field | Type | Unit | Frame | Rate | Reset | Notes |
|---|---|---|---|---|---|---|
| `wp_count` | uint16 | N/A | N/A | mission-load | HARDCODED 0 | 当前任务航点总数;init 时无任务 |
| `wp_current_index` | uint16 | N/A | N/A | mission-progress (≈ FMS 20 ms) | HARDCODED 0 | 当前正在执行的航点 idx;reset 行为见 [A6 §5 第 5 项 PRESERVED 类别](A6-init-reset-contract.md) |
| `wp_array[]` | struct array (length B1 to confirm; Phase 2 上限由 D5 锁定) | per-field | per-field | mission-load | HARDCODED zero-mission | 航点数组;每个 wp 含 `lat_deg` / `lon_deg` / `alt_m` / `cmd_type` / `param[]` 等 |
| `home_lat_deg` | float64 | deg | N/A | mission-set | PARAM(初值由 GCS 命令 set_home 注入) | home;部分实现下作为 mission 一部分 |
| `home_lon_deg` | float64 | deg | N/A | mission-set | PARAM | |
| `home_alt_m` | float32 | m | N/A | mission-set | PARAM | |
| `valid` | bool | N/A | N/A | mission-load | HARDCODED false | |
| `timestamp` | uint32 | ms | N/A | mission-load | ZERO | |

注:`wp_array` 的最大长度是契约级**必须冻结值**(避免 variable-size signal,架构 v1 §4.5);D5 多旋翼 leaf 决定数值,本文件给类别。

#### 4.4.4 FMS 输入:`INS_Out_Bus`(消费侧契约)

INS 由 firmware 拥有(架构 v1 §3 / §12.3);本表是 FMS / Controller **消费侧的字段类别清单**。`INS_Out_Bus` 完整字段表由 [B5 INS_Out_Bus 镜像同步流程](../B-contracts/B5-ins-bus-mirror.md) 在镜像产物中锁定;A3 仅给出 FMS 消费侧依赖矩阵。

边界:RB-04(INS 10 ms → FMS 20 ms,sample-time-based downsample;A4 §4.4 RB-04)。

| Field | Type | Unit | Frame | Rate | Reset | Notes |
|---|---|---|---|---|---|---|
| `position_lla` | float64[3] (lat_deg, lon_deg, alt_m) | mixed | N/A | 10 ms (INS, fw) | INS firmware-owned (model 不约束) | 大地坐标位置;经纬度专项 deg + alt m |
| `position_ned_m[3]` | float32[3] | m | NED | 10 ms | INS-owned | 本地 NED 位置;FMS gate `flag.position_valid` |
| `velocity_ned_mps[3]` | float32[3] | m/s | NED | 10 ms | INS-owned | NED 速度;FMS gate `flag.velocity_valid` |
| `quat_ned_to_b[4]` | float32[4] | N/A | NED→body | 10 ms | INS-owned | 姿态四元数;FMS gate `flag.attitude_valid` |
| `euler_ned_to_b_rad[3]` *(or firmware-legacy `euler_rad[3]`)* | float32[3] | rad | NED→body | 10 ms | INS-owned | (yaw, pitch, roll);firmware-legacy 字段名待 B5 verify |
| `ang_rate_b_radps[3]` | float32[3] | rad/s | body | 10 ms | INS-owned | filtered 角速度 |
| `acc_b_mps2[3]` | float32[3] | m/s² | body | 10 ms | INS-owned | filtered 比力 |
| `INS_Status` (`status` bitfield 或 enum) | per firmware (B5 to mirror) | N/A | N/A | 10 ms | INS-owned (HARDCODED not-ready-equiv at firmware boot) | INS 总体状态;FMS init 后必须 gate `ready` 位(per A6 §4.5.3) |
| `INS_Flag` (`flag` bitfield) | per firmware (B5 to mirror) | N/A | N/A | 10 ms | INS-owned | 含 `position_valid` / `velocity_valid` / `attitude_valid` / `mag_valid` / `gps_valid` 等位;FMS 按位 gate |
| `timestamp` | uint32 | ms | N/A | 10 ms | INS-owned ZERO at boot | per A7 |

FMS 消费侧 dependency 矩阵(per A1 §5.1.6 / A6 §4.5.3):

| FMS 子模块 | 依赖的 `INS_Out_Bus` 字段 | gate 位 |
|---|---|---|
| Mode Manager(D3) | `INS_Status.ready`、`flag.attitude_valid`、`flag.position_valid` | 所有 gate;ready=0 时停留 disarmed/idle |
| Safety Monitor(D2) | `flag.gps_valid`、`flag.position_valid`、`timestamp` (staleness) | gate gps;由 `Stale_Detector`(A8 §4.2.5)消费 timestamp |
| Mission/Auto Manager | `position_lla`、`position_ned_m`、`flag.position_valid` | gate position |
| Source Selector | (none) | |
| Command Shaper(D4) | `velocity_ned_mps`、`quat_ned_to_b`、`flag.velocity_valid`、`flag.attitude_valid` | gate vel / att |
| Output Assembler | (passthrough some validity to `FMS_Out_Bus`,详见 §4.6 / §4.7) | |

完整 fallback 行为留 [F3](../F-ins-contract/F3-consumption-rules.md) 锁定。

#### 4.4.5 FMS 输入:`Control_Out_Bus`(optional / transitional;闭合 A1 §8.3 / OQ4 的 bus 边界部分)

启用条件:**transitional**(架构 v1 §17 OQ4;A1 §4.3 OQ4 "Closed by D1+A3";D1 给功能裁决,A3 仅给 bus 边界级记录)。

边界:RB-04 等价(Controller 5 ms → FMS 20 ms,sample-time-based downsample);**仅当 D1 决议启用时存在**。

字段集(若启用):passthrough echo of `Control_Out_Bus`(per §4.3.2)。FMS 不修改其内容,仅可能消费 `motor_cmd[]` / `actuator_cmd[]` 用于:

- detection of "controller is producing zero output"(safety cross-check)
- logging / diagnostic only

**A3 决议**(per A1 §8.3 任务 brief):**A3 在 bus 边界级记录此输入存在性的开关条件,但功能上的 yes/no 决策权留给 D1。** D1 在 §3 依赖中引用本节作为 bus-level stable input。

可能启用模式枚举(给 D1 选择,A3 不裁决):

1. **永久启用**(契约保留):`Control_Out_Bus` 始终是 FMS 输入;FMS 内部对其消费/不消费由 mode 决定。
2. **过渡期启用**(transitional):Phase 2 启用以保持 firmware 行为兼容;Phase 5 / 后续移除。
3. **永久不启用**:从 codegen 输出中消除该 input port(需 codegen 配置 + interface 层调整)。

D1 选 1 / 2 / 3 中之一;A3 不预设。

#### 4.4.6 FMS 输出:`FMS_Out_Bus`

边界:RB-05 ZOH + Latch(A4 §4.4 RB-05)。**Reset-time 类别表见 A6 §4.4.2**(本文件不重复每字段类别,只在 Notes 列引用)。

`FMS_Out_Bus` 字段(架构 v1 §5.1.5 列举关键字段;本文件按类别分组并给 attributes):

##### 4.4.6.1 setpoint 命令组(被 cmd_mask 守卫)

| Field | Type | Unit | Frame | Rate | Reset | Notes |
|---|---|---|---|---|---|---|
| `pos_cmd_ned_m[3]` | float32[3] | m | NED | 20 ms | ZERO(per A6 §4.4.2 "速率/姿态/速度/加速度命令字段 ZERO") | gated by `cmd_mask.position_loop` |
| `vel_cmd_ned_mps[3]` | float32[3] | m/s | NED | 20 ms | ZERO | gated by `cmd_mask.velocity_loop` |
| `acc_cmd_ned_mps2[3]` | float32[3] | m/s² | NED | 20 ms | ZERO | gated by `cmd_mask.acceleration_loop` |
| `att_cmd_quat[4]` | float32[4] | N/A | NED→body | 20 ms | ZERO (identity quat by default; B1 to confirm 是否 [1,0,0,0] 或 ZERO 字面量) | gated by `cmd_mask.attitude_loop` |
| `att_cmd_euler_rad[3]` *(alt to quat; firmware-legacy)* | float32[3] | rad | NED→body | 20 ms | ZERO | gated by `cmd_mask.attitude_loop`;与 quat 二选一,B1 锁定 |
| `ang_rate_cmd_b_radps[3]` | float32[3] | rad/s | body | 20 ms | ZERO | gated by `cmd_mask.rate_loop` |
| `yaw_cmd_rad` | float32 | rad | NED→body | 20 ms | ZERO | gated by `cmd_mask.yaw_loop` |
| `yaw_rate_cmd_radps` | float32 | rad/s | NED→body | 20 ms | ZERO | gated by `cmd_mask.yaw_rate_loop` |
| `throttle_cmd` | float32 | normalized 0..1 | N/A | 20 ms | ZERO | gated by `cmd_mask.throttle_passthrough` |

##### 4.4.6.2 控制流 / 状态组

| Field | Type | Unit | Frame | Rate | Reset | Notes |
|---|---|---|---|---|---|---|
| `cmd_mask` | uint32 bitfield | N/A | N/A | 20 ms | HARDCODED zero(safe state per A6 §4.4.2)| Latch RB-05;**位语义由 D6 锁定**;Controller 据此裁剪环路 |
| `ctrl_mode` | enum (CtrlMode per A2 R-6.2; B2 to lock) | N/A | N/A | 20 ms | HARDCODED disarmed/idle-equiv | Controller 据此选择控制律分支 |
| `mode` | enum (FlightMode/PilotMode 系列) | N/A | N/A | 20 ms | HARDCODED disarmed/idle-equiv | 由 D3 状态机锁定 |
| `status` | enum (VehicleStatus per A2 R-6.2) | N/A | N/A | 20 ms | HARDCODED disarmed-equiv | |
| `state` | enum (VehicleState) | N/A | N/A | 20 ms | HARDCODED disarmed-equiv | |
| `ext_state` | enum | N/A | N/A | 20 ms | HARDCODED disarmed-equiv | extended state |
| `reset` *(optional, per A6 §5 第 2 项 open)* | bool | N/A | N/A | 20 ms | ZERO | global reset 触发位;由 D3 / D2 safety 在 step 内置位;Latch RB-05 |

##### 4.4.6.3 任务 / 导航 / 错误组

| Field | Type | Unit | Frame | Rate | Reset | Notes |
|---|---|---|---|---|---|---|
| `wp_index` | uint16 | N/A | N/A | 20 ms | HARDCODED 0 (or PRESERVED if A6 §5 PRESERVED applies) | 当前 wp 索引;passthrough from `Mission_Data_Bus.wp_current_index` |
| `wp_lat_deg` | float64 | deg | N/A | 20 ms | ZERO | 当前 wp 经纬度(快照)|
| `wp_lon_deg` | float64 | deg | N/A | 20 ms | ZERO | |
| `wp_alt_m` | float32 | m | N/A | 20 ms | ZERO | |
| `home_lat_deg` | float64 | deg | N/A | 20 ms | PARAM (echo `Mission_Data_Bus.home_lat_deg`) | |
| `home_lon_deg` | float64 | deg | N/A | 20 ms | PARAM | |
| `home_alt_m` | float32 | m | N/A | 20 ms | PARAM | |
| `error_code` | enum (ErrorCode per A2 R-6.2; B2 to lock) | N/A | N/A | 20 ms | HARDCODED no-error-equiv | Safety Monitor 推出 |
| `failsafe_state` *(optional)* | enum | N/A | N/A | 20 ms | HARDCODED none-equiv | failsafe 子状态 |

##### 4.4.6.4 actuator passthrough 组(闭合 A1 §7.2)— passthrough field set

架构 v1 §7.2 / §5.1.5 给出 "actuator_cmd passthrough space",意为 FMS 不**生成**这些字段,而是**直传** Pilot/GCS/Auto 来源中已有的相应执行器命令(通常用于 manual / passthrough mode 下绕过 Controller 内环)。

**Passthrough field set 枚举**(best-effort,基于架构 v1 §5.1.5 + A1 衍生 + 多旋翼 firmware mc_fms 行为):

| Field | Type | Unit | Frame | Rate | Reset | Notes (passthrough source) |
|---|---|---|---|---|---|---|
| `actuator_cmd[]` | float32 array | normalized 或 trim | mixed | 20 ms | ZERO | passthrough from `Pilot_Cmd_Bus` 杆位 (manual mode) 或 `Auto_Cmd_Bus.cmd_param` (auto override) |
| `motor_cmd_passthrough[]` *(optional, manual passthrough mode)* | float32 array | normalized | N/A | 20 ms | ZERO | 直传电机命令,绕过 Controller 内环;variant-conditional;多旋翼默认禁用 |
| `pilot_throttle_passthrough` | float32 | normalized | N/A | 20 ms | ZERO | passthrough from `Pilot_Cmd_Bus.throttle_stick_n01`;manual mode 启用 |
| `pilot_yaw_rate_passthrough_radps` *(optional)* | float32 | rad/s | NED→body | 20 ms | ZERO | passthrough from yaw stick × shaper;某些 mode 下作为 ang_rate_cmd 的预合并源 |

**A3 决议**(闭合 A1 §7.2):

- **生成字段**(FMS 自己计算):`pos_cmd` / `vel_cmd` / `acc_cmd` / `att_cmd` / `ang_rate_cmd` / `yaw_cmd` / `yaw_rate_cmd` / `throttle_cmd`(setpoint 组)、`cmd_mask` / `ctrl_mode` / `mode` / `status` / `state` / `ext_state` / `reset` / `error_code` / `failsafe_state`(状态组)、`wp_index` / `wp_lat_deg` / `wp_lon_deg` / `wp_alt_m` / `home_*`(导航组,部分 echo from Mission_Data_Bus,部分 FMS 计算)。
- **Passthrough 字段**(FMS **不**生成,直传或 echo upstream):`actuator_cmd[]`、`motor_cmd_passthrough[]`、`pilot_throttle_passthrough`、`pilot_yaw_rate_passthrough_radps`、`wp_lat/lon/alt_deg/m`(部分:从 `Mission_Data_Bus.wp_array[wp_current_index]` 抽取后透传)、`home_lat/lon/alt_*`(从 `Mission_Data_Bus` 直传)。
- 完整逐字段判定由 [D2](../D-fms/D2-fms-structural.md) Output Assembler 子模块在自身退出条件中复核;A3 给出**类别层**枚举。

注:架构 v1 §5.1.5 只口径表述"actuator_cmd passthrough space",字段集的精确成员由 firmware `FMS_types.h` 锁定,**byte 级由 B1 verify**。

##### 4.4.6.5 INS validity passthrough(闭合 §7.2 衍生 — A1 §5.1.6 controller 输入字段方向)

`FMS_Out_Bus` 中可能存在 INS validity 的部分 echo,以便 Controller 不需直接消费 `INS_Out_Bus.flag` 全字段:

| Field | Type | Unit | Frame | Rate | Reset | Notes |
|---|---|---|---|---|---|---|
| `ins_ready` *(optional echo)* | bool | N/A | N/A | 20 ms | HARDCODED false | passthrough echo from `INS_Out_Bus.INS_Status.ready`;Controller gate 依赖 |
| `ins_position_valid` *(optional echo)* | bool | N/A | N/A | 20 ms | HARDCODED false | passthrough echo from `INS_Out_Bus.flag.position_valid` |
| `ins_attitude_valid` *(optional echo)* | bool | N/A | N/A | 20 ms | HARDCODED false | passthrough echo from `INS_Out_Bus.flag.attitude_valid` |

A3 决议:这些 echo 字段的存在与否由 B1 锁定;若 firmware `FMS_Out_Bus` 含此类字段,A3 在此声明它们是 **passthrough** 类别;若 firmware 不含,Controller 必须直接消费 `INS_Out_Bus.flag`(契约级双消费,这是默认设计)。

##### 4.4.6.6 timestamp

| Field | Type | Unit | Frame | Rate | Reset | Notes |
|---|---|---|---|---|---|---|
| `timestamp` | uint32 | ms | N/A | 20 ms | ZERO | per A7 §4.6;**模型仓内部不重新打戳**,直接使用 step interface 入参 timestamp |

### 4.5 Controller 模块边界

Controller 在模型仓侧周期 = **5 ms**(架构 v1 §13)。Controller 输入 / 输出契约见架构 v1 §5.1.6 / §8.4。

#### 4.5.1 Controller 输入/输出 bus 总览

| 方向 | Bus | 来源 / 去向 | mandatory / optional / variant-conditional | 跨速率边界 |
|---|---|---|---|---|
| Input | `FMS_Out_Bus` | FMS (20 ms) → Controller (5 ms) | **mandatory** | RB-05 ZOH + Latch (A4 §4.4 RB-05) |
| Input | `INS_Out_Bus` | INS (firmware) / `ins_stub` (MIL) | **mandatory** | RB-03 ZOH (A4 §4.4 RB-03) |
| Input | `Plant_States_Bus` | Plant (1 ms) → Controller (5 ms) | **variant-conditional**(harness/SIH/HIL only;per [A5 §4.5](A5-variant-strategy.md) + 架构 v1 §8.4)| RB-01 sample-time-based downsample (仅 variant) |
| Output | `Control_Out_Bus` | Controller (5 ms) → Plant (1 ms) + (optional) FMS | **mandatory** | RB-02 ZOH (A4 §4.4 RB-02);若启用 FMS 回送,亦走 RB-04 等价 |

#### 4.5.2 Controller 输入:`FMS_Out_Bus`

字段同 §4.4.6(同一 bus,Controller 是消费方)。Controller 消费侧 dependency:

| Controller 子模块 | 依赖的 `FMS_Out_Bus` 字段 | 守卫 |
|---|---|---|
| Setpoint Decode(架构 v1 §12.2.1) | `cmd_mask`、`ctrl_mode`、所有 setpoint 字段 | Latch RB-05 整 bus 同帧锁存 |
| Position Loop(可选,§12.2.2) | `pos_cmd_ned_m` | gated `cmd_mask.position_loop` |
| Velocity Loop | `vel_cmd_ned_mps` | gated `cmd_mask.velocity_loop` |
| Attitude Loop | `att_cmd_quat` 或 `att_cmd_euler_rad` | gated `cmd_mask.attitude_loop` |
| Rate Loop | `ang_rate_cmd_b_radps` | gated `cmd_mask.rate_loop` |
| Yaw / Yaw-rate | `yaw_cmd_rad`、`yaw_rate_cmd_radps` | gated `cmd_mask.yaw_loop` / `cmd_mask.yaw_rate_loop` |
| Mixer / Allocator(§12.2.6) | `throttle_cmd`、各 setpoint 经过环路后的虚拟控制 | (no extra gate) |
| Output Assembler(§12.2.7) | `reset`(global reset 信号)、`ctrl_mode` | reset → Controller local reset per A6 §4.5.2 |
| INS readiness gate | `ins_ready` echo(若存在,§4.4.6.5)否则直接消费 `INS_Out_Bus.flag` | by D6/E1 |

#### 4.5.3 Controller 输入:`INS_Out_Bus`

字段同 §4.4.4(同一 bus)。边界 RB-03(10 ms → 5 ms ZOH)。Controller 消费侧 dependency:

| Controller 子模块 | 依赖的 `INS_Out_Bus` 字段 | gate 位 |
|---|---|---|
| Position Loop | `position_ned_m`、`flag.position_valid` | gate position |
| Velocity Loop | `velocity_ned_mps`、`flag.velocity_valid` | gate velocity |
| Attitude Loop | `quat_ned_to_b` 或 `euler_ned_to_b_rad`、`flag.attitude_valid` | gate attitude |
| Rate Loop | `ang_rate_b_radps`(filtered by INS) | (intrinsic;角速率环依赖 INS filtered ω,不消费 raw IMU) |
| (内环加速度 / mass / battery 校正) | `acc_b_mps2` | optional |

注:Controller 不直接消费 `INS_Out_Bus.timestamp`(per A7 §4.5 dt 由固定步长得来);但跨速率 staleness 检测(若 E1 / D6 决定加入)由 `Stale_Detector`(A8 §4.2.5)消费 timestamp。

#### 4.5.4 Controller 输入:`Plant_States_Bus`(variant-conditional)

字段同 §4.3.5。**仅 harness/SIH/HIL variant 启用**(A5 §4.5)。Phase 2 MIL 默认禁用。

#### 4.5.5 Controller 输出:`Control_Out_Bus`

字段同 §4.3.2(同一 bus,Controller 是产出方)。

补充字段类别(A3 视角,Controller 侧):

| Field | Type | Unit | Frame | Rate | Reset | Notes |
|---|---|---|---|---|---|---|
| `motor_cmd[]` | float32 array | normalized 0..1 或 PWM `_us`(per E4) | N/A | 5 ms | HARDCODED zero-thrust (A6 §4.4.2) | Controller 经 mixer 计算 |
| `actuator_cmd[]` | float32 array | normalized 或 trim | body | 5 ms | HARDCODED trim | per E4 leaf |
| `throttle_cmd` *(echo)* | float32 | normalized 0..1 | N/A | 5 ms | ZERO | passthrough from `FMS_Out_Bus.throttle_cmd` 当 throttle_passthrough mode |
| `cmd_mask` *(echo)* | uint32 | N/A | N/A | 5 ms | HARDCODED zero | passthrough from `FMS_Out_Bus.cmd_mask`;Controller 不修改,Plant trace 用 |
| `ctrl_mode` *(echo)* | enum | N/A | N/A | 5 ms | HARDCODED disarmed-equiv | passthrough |
| `reset` *(echo, optional)* | bool | N/A | N/A | 5 ms | ZERO | passthrough from `FMS_Out_Bus.reset` |
| `timestamp` | uint32 | ms | N/A | 5 ms | ZERO | per A7;直接使用 step interface 入参 |

### 4.6 FMS 输出 passthrough 字段集(集中枚举,闭合 A1 §7.2)

为方便 D2 Output Assembler 与 B4 contract diff 识别,把 §4.4.6 中标 passthrough 的字段汇总:

| `FMS_Out_Bus` 字段 | passthrough 源 bus | passthrough 源字段 | 转化 | A3 类别 |
|---|---|---|---|---|
| `actuator_cmd[]` | `Pilot_Cmd_Bus` / `Auto_Cmd_Bus` | sticks / cmd_param | 经 D4 Command Shaper 整形(死区 + 限幅 + 单位换算) | passthrough-with-shape |
| `motor_cmd_passthrough[]` *(optional)* | `Pilot_Cmd_Bus` / `Auto_Cmd_Bus` | manual mode 下指定 | 直传(无 shaper) | passthrough-direct |
| `pilot_throttle_passthrough` | `Pilot_Cmd_Bus.throttle_stick_n01` | direct | 经 D4 死区 / 单位换算 | passthrough-with-shape |
| `pilot_yaw_rate_passthrough_radps` | `Pilot_Cmd_Bus.yaw_stick_n01` | direct | 经 D4 stick→rate 映射(含 PARAM 增益) | passthrough-with-shape |
| `wp_lat_deg` / `wp_lon_deg` / `wp_alt_m` | `Mission_Data_Bus.wp_array[wp_current_index]` | wp.lat_deg / lon_deg / alt_m | 由 mission manager 选取并 echo | passthrough-direct |
| `home_lat_deg` / `home_lon_deg` / `home_alt_m` | `Mission_Data_Bus.home_*` 或 `GCS_Cmd_Bus` set_home | direct | echo | passthrough-direct |
| `ins_ready` *(optional echo)* | `INS_Out_Bus.INS_Status.ready` | direct | echo | passthrough-direct |
| `ins_position_valid` *(optional echo)* | `INS_Out_Bus.flag.position_valid` | direct | echo | passthrough-direct |
| `ins_attitude_valid` *(optional echo)* | `INS_Out_Bus.flag.attitude_valid` | direct | echo | passthrough-direct |

注:`passthrough-with-shape` 表示 FMS 经过 [`FMS_Command_Shaper_Shell`(A8 §4.3.2)](A8-shared-library-roster.md) 做单位换算 / 死区 / 限幅,不改变信号"来源"语义;`passthrough-direct` 表示 FMS 仅做选取或位拷贝。完整闭合 A1 §7.2 缺口(架构 v1 §7.2 "Does not own ... beyond any contract-required passthrough fields" passthrough 字段集合未枚举)。

### 4.7 cmd_mask 守卫规则(指针 — 由 D6 + E1 锁定)

A3 在每个 setpoint 字段的 Notes 列已标 "gated by cmd_mask.<X>",但**位语义本身**(哪一位代表哪个环路)由 [D6 FMS↔Controller 接口约定](../D-fms/D6-fms-controller-interface.md) 与 [E1 Controller 功能设计](../E-controller/E1-controller-functional.md) 协同 batch(co-seal,Wave 8)锁定。A3 仅承诺:

- `cmd_mask` 是 `FMS_Out_Bus` 中 uint32 bitfield(passthrough 到 `Control_Out_Bus`);
- A3 标记的 `cmd_mask.position_loop` / `velocity_loop` / `acceleration_loop` / `attitude_loop` / `rate_loop` / `yaw_loop` / `yaw_rate_loop` / `throttle_passthrough` 是工作名,具体位号 / 名 由 D6 锁定(B2 锁 enum 数值);
- 同一帧 Latch RB-05 保证 cmd_mask 与对应 setpoint **同帧到达** Controller。

### 4.8 INS 消费侧依赖矩阵(集中表;闭合 A1 §5.1.6)

A1 §5.1.6 缺口 "Controller Inputs/Outputs 仅列 bus 名,未给字段方向矩阵" — A3 给出的"FMS / Controller 各子模块对 INS_Out_Bus 字段依赖"集中如下:

| `INS_Out_Bus` 字段 | FMS Mode Mgr | FMS Safety | FMS Mission/Auto | FMS Source Sel | FMS Shaper | FMS Output Asm | Ctrl Pos | Ctrl Vel | Ctrl Att | Ctrl Rate | Ctrl Mixer |
|---|---|---|---|---|---|---|---|---|---|---|---|
| `position_lla` | — | — | ● | — | — | — | — | — | — | — | — |
| `position_ned_m` | gate | — | ● | — | — | — | ● | ●* | — | — | — |
| `velocity_ned_mps` | — | — | — | — | ● | — | — | ● | — | — | — |
| `quat_ned_to_b` | gate | — | — | — | ● | — | — | — | ● | — | — |
| `euler_ned_to_b_rad` | — | — | — | — | (alt) | log | — | — | (alt) | — | — |
| `ang_rate_b_radps` | — | — | — | — | — | — | — | — | — | ● | — |
| `acc_b_mps2` | — | — | — | — | — | — | — | (opt) | — | — | — |
| `INS_Status.ready` | gate | gate | gate | — | — | echo* | — | — | — | — | — |
| `flag.position_valid` | gate | gate | gate | — | — | echo* | gate | gate* | — | — | — |
| `flag.velocity_valid` | — | — | — | — | gate | — | — | gate | — | — | — |
| `flag.attitude_valid` | gate | — | — | — | gate | echo* | — | — | gate | — | — |
| `flag.gps_valid` | — | gate | — | — | — | — | — | — | — | — | — |
| `flag.mag_valid` | — | gate | — | — | — | — | — | — | — | — | — |
| `timestamp` | — | gate(stale) | — | — | — | — | — | — | — | — | — |

记号:`●` 直接消费;`gate` 用作守卫;`(opt)` 可选消费(由各下游设计文件决定);`(alt)` 与 quat 二选一(代替源);`*` 表示该消费经由 `FMS_Out_Bus` echo 字段间接到达(若 §4.4.6.5 echo 字段存在)。`log` 表示仅日志使用。

详细 fallback 行为留 [F3](../F-ins-contract/F3-consumption-rules.md) 锁定。

### 4.9 字段方向总表(per bus 总结)

下表是 §4.3 / §4.4 / §4.5 各 bus 表的总览,供下游(B1 / B4 / G1 / G3)做对照:

| Bus | Direction(模型仓视角) | 产出方周期 | 主消费方 | 字段类型/属性源 | 字段是否 firmware-legacy 非默认 |
|---|---|---|---|---|---|
| `Pilot_Cmd_Bus` | Input → FMS | external (≈ 50–100 Hz) | FMS Source Selector | A3 §4.4.2 | mixed(stick 字段沿 firmware-legacy) |
| `GCS_Cmd_Bus` | Input → FMS | external (≈ 5–20 Hz) | FMS Source Selector | A3 §4.4.3.1 | mixed |
| `Auto_Cmd_Bus` | Input → FMS | external | FMS Source Selector / Mission | A3 §4.4.3.2 | mixed |
| `Mission_Data_Bus` | Input → FMS | mission-load | FMS Mission/Auto Manager | A3 §4.4.3.3 | mixed |
| `INS_Out_Bus` | Input → FMS, Controller | 10 ms (firmware-owned) | FMS / Controller (multi-consumer) | A3 §4.4.4 + §4.5.3 + §4.8 矩阵 | yes(由 firmware 锁定;B5 镜像) |
| `Control_Out_Bus` | Output ← Controller; Input → Plant; (optional Input → FMS per §4.4.5) | 5 ms | Plant (主), FMS (回送可选) | A3 §4.3.2 / §4.5.5 | yes |
| `FMS_Out_Bus` | Output ← FMS; Input → Controller | 20 ms | Controller | A3 §4.4.6 | yes(关键契约) |
| `Plant_States_Bus` | Output ← Plant; Input → ins_stub (MIL) / Controller (variant) | 1 ms | ins_stub (主), Controller (variant) | A3 §4.3.5 | mixed |
| `Extended_States_Bus` | Output ← Plant | 1 ms | harness / observer | A3 §4.3.6 | mixed |
| `Environment_Info_Bus` | Input → Plant | configuration | Plant | A3 §4.3.3 | mixed |
| `States_Init_Bus` | Input → Plant | configuration | Plant init | A3 §4.3.4 | mixed |
| `IMU_Bus` | Output ← Plant; Input → ins_stub | 1 ms (typ.) | ins_stub | A3 §4.3.7.1 | mixed |
| `MAG_Bus` | Output ← Plant; Input → ins_stub | per C1 | ins_stub | A3 §4.3.7.2 | mixed |
| `Barometer_Bus` | Output ← Plant; Input → ins_stub | per C1 | ins_stub | A3 §4.3.7.3 | mixed |
| `GPS_uBlox_Bus` | Output ← Plant; Input → ins_stub | per C1 | ins_stub | A3 §4.3.7.4 | mixed |
| `AirSpeed_Bus` | Output ← Plant; Input → ins_stub (variant) | per C1 | ins_stub (FW/VTOL only) | A3 §4.3.7.5 | mixed; variant-conditional |

**16 / 16 firmware-bus 全覆盖**(任务 brief 验收清单)。

### 4.10 OQ4 / A1 §8.3 处置(`Control_Out_Bus` 作为 FMS 输入)

闭合 A1 §4.3 OQ4 / §4.2 §8.3 缺口的 **bus 边界级**部分:

A3 决议:

1. **A3 给 bus 边界**:`Control_Out_Bus` **可能**作为 FMS 输入存在;若启用,字段集 = §4.3.2(同 `Control_Out_Bus`),边界 = RB-04 等价(5 ms → 20 ms sample-time-based downsample),Reset-time 类别同 §4.3.2(passthrough 类),不重复列。
2. **A3 不裁决功能 yes/no**:启用 / 禁用 / 过渡 三选一由 [D1 FMS 功能设计](../D-fms/D1-fms-functional.md) 在自身退出条件中决定。
3. **A3 给 D1 一个稳定输入**:无论 D1 选择启用与否,A3 这一节描述的"`Control_Out_Bus` 字段语义 / Reset 类别 / 边界 ID"在 bus 边界级**永远不变**;D1 只决定该 bus 是否进入 FMS interface 函数签名。
4. **若 D1 选"启用"**:A3 §4.4.1 的总览表在"FMS 输入"列保留 `Control_Out_Bus` 行,标 mandatory;§4.4.5 的 optional 标签升级为 mandatory。
5. **若 D1 选"过渡"**:A3 §4.4.1 保留 optional;Phase 5 / 后续移除时由 D1 触发本文件变更日志。
6. **若 D1 选"永久不启用"**:A3 §4.4.1 该行删除,触发 codegen 配置(I4) + interface 层调整。

记此条入 §5 risk(B1 / D1 协同)。

### 4.11 与 firmware 路径的对称性约束

按 A6 §4.7.2 对称性保留:在 firmware 路径下,Plant 是否被链接由 SIH/HIL 配置决定(A6 §5 第 3 项 open);A3 在 bus 边界级:

- `Control_Out_Bus` / `FMS_Out_Bus` / `INS_Out_Bus`(契约 bus)字段集与 reset 类别在两条路径下**完全等价**;
- `Plant_States_Bus` / `Extended_States_Bus` / 各传感器 bus 仅 MIL / SIH 出现,firmware 飞行运行期不存在(架构 v1 §3 / §15.4);
- harness-only 字段 / 信号(若 G1 在 harness Variant 中加 trace 信号)**不**进入 firmware export(per [A5 §4.6.4](A5-variant-strategy.md))。

### 4.12 字段元数据要求(传递给 B1)

A3 要求 B1 字段表必须包含的元数据列:

| 列 | 来源(本文件) |
|---|---|
| 字段名 | A3 §4.x 表 |
| 类型 | A3 §4.x Type 列(若 A3 标 "B1 to confirm",B1 必须 byte-verify) |
| 单位后缀 | A3 §4.x Unit 列 |
| 坐标系后缀 | A3 §4.x Frame 列 |
| 产出方 nominal rate | A3 §4.x Rate 列 |
| reset-time 稳态值类别 | A3 §4.x Reset 列(per A6 §4.4) |
| variant-conditional 标记 | A3 Notes 列;A5 §4.2.2 |
| firmware-legacy 字段名标记 | A3 Notes 列;A2 E-5.1 |
| init_use 标志 | per A6 §4.6.2(若字段被 init 使用,B3 标注) |

B1 在 byte 级锁定时若发现 A3 字段名 / 类型与 firmware 不一致,以 firmware 为准并触发 A3 §8 变更日志。

## 5. 已知风险与悬而未决问题

- **firmware commit hash 暂缺**
  - 影响:RULES §5 / §3 表 require 镜像自 firmware 的契约记录 commit hash + 文件相对路径;本文件 §3 仅给路径占位 `FMT-Firmware @ <pending hash>`。
  - 处置:open;评审通过前由 reviewer / orchestrator 在 INDEX 决策日志登记本工作项时补齐(per [A6 §5 第 1 项](A6-init-reset-contract.md) 同样的处置约定)。本工作项 contract_impact=yes 已声明。

- **多个字段标 "B1 to confirm"**
  - 影响:`Control_Out_Bus.motor_cmd[]` 单位(normalized vs PWM_us);`MAG_Bus.mag_b_*` 单位(Gauss vs nT);四元数顺序(w,x,y,z 还是 x,y,z,w);euler 顺序(ZYX 还是其他);`Mission_Data_Bus.wp_array` 长度;`actuator_cmd[]` 含义(normalized vs trim);`att_cmd_quat` reset 是否字面量 [1,0,0,0] 或 ZERO 字面量;`euler_rad` 字段名是否带 `_ned_to_b_` 中缀。
  - 处置:open;由 [B1](../B-contracts/B1-bus-inventory.md) byte-verify firmware `*_types.h` 后回填;A3 §8 变更日志记录每条回填。
  - 监控点:本工作项与 B1 协同,B1 退出条件须**显式**对每条 "B1 to confirm" 给出回填或拒接(若 firmware 无对应字段)。

- **`FMS_Out_Bus.reset` 字段是否真的存在**
  - 影响:§4.4.6.2 / §4.10 假设存在;[A6 §5 第 2 项 open](A6-init-reset-contract.md) 同样的 open。
  - 处置:同 A6;由 B1 锁定。若不存在,A3 §4.4.6.2 行删除并改用 cmd_mask + ctrl_mode 派生 reset 信号(D6 / D2 协同处理)。

- **passthrough 字段集枚举的完整性**
  - 影响:§4.4.6.4 + §4.6 给出的 passthrough 集合是 best-effort,基于架构 v1 §5.1.5 + multicopter mc_fms 模式推断。若 firmware `FMS_Out_Bus` 含未覆盖的 passthrough 字段(例如某些机型独有的 `gimbal_cmd` 等),A3 当前清单不涵盖。
  - 处置:open;由 D2 Output Assembler 在结构设计中复核,B1 字节验证时反馈;触发本文件变更日志。

- **`Control_Out_Bus` 是否含 `cmd_mask` / `ctrl_mode` / `reset` echo**
  - 影响:§4.3.2 / §4.5.5 假设这些 echo 字段存在(用于 Plant trace + Plant 不解释);若 firmware `Control_Out_Bus` 不含,这些行删除,Controller→Plant 边界仅传执行器命令。
  - 处置:open;由 B1 锁定;不影响 A3 §4.5.5 motor_cmd / actuator_cmd / throttle_cmd 主字段集。

- **FMS validity echo(`ins_ready` / `ins_position_valid` 等)是否存在**
  - 影响:§4.4.6.5 — 若 firmware `FMS_Out_Bus` 不含,Controller 必须直接消费 `INS_Out_Bus.flag`,§4.8 dependency 矩阵的 `*` 标记需移除。
  - 处置:open;由 B1 锁定;default 假设是 echo 不存在(契约更小,Controller 双消费),A3 中以 *(optional echo)* 标。

- **OQ4 在 D1 决策前 A3 不能锁 §4.10**
  - 影响:D1 在 Wave 6 启动;在 D1 完成前,A3 §4.4.5 / §4.10 标 transitional;D1 选项 1 / 2 / 3 决定后,A3 走变更日志同步。
  - 处置:open;A3 在初稿即给出 D1 三选项 stable input,等待 D1 裁决。

- **variant-conditional 字段的 codegen 形态**
  - 影响:§4.3.7.5 `AirSpeed_Bus`、§4.3.6 `aoa_rad` / `aos_rad` / `flow_angle_b_rad`、§4.4.6.4 `motor_cmd_passthrough[]`、§4.4.5 `Control_Out_Bus` 整 bus 等是 variant-conditional。Phase 2 多旋翼下这些字段 / bus 的存在性由 [I4 codegen 配置](../I-tooling/I4-codegen-config.md) + [A5 §4.6](A5-variant-strategy.md) Variant Subsystem "Single design" 配置决定。
  - 处置:open;A3 仅标 variant-conditional;具体生成 / 不生成由 I4 + A5 落地。

- **慢传感器 cadence 数值未定**
  - 影响:§4.3.7 各传感器 bus 的 Rate 列引用 "per C1";C1 未启动(Wave 6),A3 暂以 typical 值给注释。
  - 处置:open;由 C1 锁定;A3 §4.3.7 走变更日志同步。

- **A3 字段表与 firmware-legacy 命名冲突**
  - 影响:A2 §4.10 R-5.3 要求字段含单位/坐标系后缀,firmware-legacy 字段不含。本文件给出字段名时倾向新规则(如 `vel_ned_mps`),但实际 firmware 可能用 `velocity_ned`。每条不同处需 B1 锁定。
  - 处置:open;A2 E-5.1 已明示 firmware 字段名优先;B1 锁字节顺序时按 firmware 实际名称回填本文件,所有 A3 表行的 Notes 列同步标 firmware-legacy。

- **本工作项 contract_impact=yes 的影响范围**
  - 影响:任何 A3 字段方向 / Reset 类别 / passthrough 标签变化 = firmware 契约风险;变更必须经 INDEX 决策日志登记。
  - 处置:已声明 contract_impact=yes;变更走 RULES §10 + INDEX 登记。本文件不**定义**字节布局,仅描述 field-level 语义,故风险面比 B1 小但仍 yes(因:A3 决定字段方向 / 类别会迫使 B1 / B2 / B3 在锁定时遵循 A3 决策)。

## 6. 退出条件复核

A3 在 [`00-design-plan.md`](../00-design-plan.md) §4.A 中的退出条件原文:

> Plant/FMS/Controller 三个模块的输入/输出 bus 字段级清单与方向定稿;**每个字段附单位与坐标系标注**

逐条复核:

| # | 退出条件分解 | 本文档依据 | 状态 |
|---|---|---|---|
| 1 | "Plant 三个模块输入/输出 bus 清单与方向" — Plant 部分 | §4.3.1 总览表(Input × 3, Output × 7);§4.3.2~§4.3.7 各 bus 字段表 | 满足 |
| 2 | "FMS 三个模块输入/输出 bus 清单与方向" — FMS 部分 | §4.4.1 总览表(Input × 6, Output × 1);§4.4.2~§4.4.6 各 bus 字段表 | 满足 |
| 3 | "Controller 三个模块输入/输出 bus 清单与方向" — Controller 部分 | §4.5.1 总览表(Input × 3, Output × 1);§4.5.2~§4.5.5 各 bus 字段表 / 引用 | 满足 |
| 4 | "字段级清单" — field 级 | §4.3 / §4.4 / §4.5 共 16 个 bus 全部以字段级表格列出 | 满足 |
| 5 | "方向定稿" | §4.3.1 / §4.4.1 / §4.5.1 总览表给方向(Input/Output);§4.9 字段方向总表汇总 | 满足 |
| 6 | "每个字段附**单位**标注" | §4.3 / §4.4 / §4.5 表中 Unit 列;空字段(布尔/enum/计数)标 N/A | 满足(per A2 §4.10.1) |
| 7 | "每个字段附**坐标系**标注" | §4.3 / §4.4 / §4.5 表中 Frame 列;无坐标系字段标 N/A | 满足(per A2 §4.10.2) |

附:对 [A1](A1-v1-review.md) §4.2 / §4.3 中归属 A3 的缺口逐条闭合:

| 来源(A1) | 缺口简述 | 本文档闭合位置 |
|---|---|---|
| A1 §4.2 §5.1.4 | FMS 输入清单 mandatory/optional + `Control_Out_Bus` 回送条件 | §4.4.1 总览表(mandatory/optional 列)+ §4.4.5 + §4.10 OQ4 处置 |
| A1 §4.2 §5.1.5 | `FMS_Out_Bus` 字段顺序与类型(actuator passthrough 等)| §4.4.6 字段表 + §4.6 passthrough 集中表(类型留 B1 byte-verify) |
| A1 §4.2 §5.1.6 | Controller Inputs/Outputs 字段方向矩阵 | §4.5 全节 + §4.8 INS 消费侧依赖矩阵 |
| A1 §4.2 §5.1.7 | Plant 输入/输出含 "optional others" 模糊列举 — A3 列定 optional 取舍判据 | §4.3.6 Extended_States_Bus optional 字段 + §4.3.7.5 AirSpeed_Bus variant-conditional |
| A1 §4.2 §7.2 | FMS passthrough 字段集合枚举 | §4.4.6.4 + §4.6(集中表)|
| A1 §4.2 §8.1 | Plant 输出 "optional others" 入选准则 | §4.3.1 总览表(每 bus 标 mandatory/optional/variant-conditional)+ §4.3.6 / §4.3.7 字段级 optional 标 |
| A1 §4.2 §8.3 | FMS optional `Control_Out_Bus` 启用条件 | §4.4.5 + §4.10(A3 给 bus 边界,功能裁决留 D1)|
| A1 §4.3 OQ4 | `Control_Out_Bus` 长期是否作为 FMS 输入 | §4.10 三选项;A3 给 stable bus boundary,D1 在 Wave 6 裁决 |

退出条件全部满足;状态待 reviewer 升级到 reviewed。

## 7. 下游影响

按 [`01-design-relationships.md §4.1 / §4.2 / §4.3 / §4.4 / §4.5 / §4.6 / §4.7 / §4.10`](../01-design-relationships.md) A3 出边:

```text
A2, A4, A5 → A3
A3 → B1, B2, B3
A3, B1 ⇢ C1
A3, B1, B2 ⇢ D1
A3, B1, B2 ⇢ E1
A3, B1 ⇢ I4
A6 ⇢ B3, C1, C4, D1, E1   (A3 间接传递 A6 类别约束)
```

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [B1 Bus 清单与 schema 设计](../B-contracts/B1-bus-inventory.md) *(未启动,Wave 4)* | → | §4.3 / §4.4 / §4.5 全部 16 个 bus 的字段级清单(类型 / 单位 / 坐标系 / Reset 类别 / 方向);§4.12 字段元数据要求;每条 "B1 to confirm" 是 B1 的 byte-verify task list |
| [B2 Enum 清单与数值锁定](../B-contracts/B2-enum-inventory.md) *(未启动,Wave 4)* | → | §4.4.6 / §4.5 中所有 enum 字段(`ctrl_mode` / `mode` / `status` / `state` / `ext_state` / `error_code` / `failsafe_state` / `mode_switch` / `cmd_type` / `fix_type`)入清单 |
| [B3 Parameter schema 设计](../B-contracts/B3-parameter-schema.md) *(未启动,Wave 4)* | → | §4.x 表 Reset 列中 PARAM 类别字段全部进入 PARAM struct schema;§4.3.4 `States_Init_Bus` 字段是 `PLANT_PARAM` init 候选;`Environment_Info_Bus` 全字段是 `PLANT_PARAM` 候选 |
| [C1 Plant 功能设计](../C-plant/C1-plant-functional.md) *(未启动,Wave 6)* | ⇢ | §4.3.6 Extended_States_Bus optional 字段集;§4.3.7 各传感器 bus cadence 由 C1 锁定 |
| [C2 Plant 结构设计](../C-plant/C2-plant-structural.md) *(未启动,Wave 7)* | ⇢ | §4.3 全节(输入 / 输出 bus 字段表)+ §4.3.6 optional 字段取舍 |
| [C3 Plant 多旋翼 leaf 设计](../C-plant/C3-multicopter-leaf.md) *(未启动,Wave 9)* | ⇢ | §4.3.5 `motor_rpm[]` 数组长度;§4.3.4 `States_Init_Bus.motor_state[]` |
| [C4 Plant 数值与积分设计](../C-plant/C4-numerics.md) *(未启动,Wave 9)* | ⇢ | §4.3.5 数值积分器对应的 Plant_States 字段(per A6 §4.4.2 行 "数值积分器内部状态") |
| [D1 FMS 功能设计](../D-fms/D1-fms-functional.md) *(未启动,Wave 6)* | ⇢ | §4.4.1 mandatory/optional 列;§4.4.5 + §4.10 OQ4 三选项;§4.4.6 setpoint 组与 mode 组;§4.6 passthrough 集中表;§4.8 INS 消费侧依赖矩阵 |
| [D2 FMS 结构设计](../D-fms/D2-fms-structural.md) *(未启动,Wave 7)* | ⇢ | §4.4.6 全表;§4.6 passthrough 集中表(指引 Output Assembler);§4.4.4 INS 消费 dependency |
| [D3 FMS Mode Manager](../D-fms/D3-mode-manager.md) *(未启动,Wave 8)* | ⇢ | §4.4.6.2 控制流字段(`ctrl_mode` / `mode` / `status` / `state` / `ext_state` / `reset`);§4.8 Mode Mgr 列 |
| [D4 FMS Command Shaper](../D-fms/D4-command-shaper.md) *(未启动,Wave 8)* | ⇢ | §4.4.6.1 setpoint 字段及其 Reset ZERO;§4.6 passthrough-with-shape 行 |
| [D5 FMS 多旋翼 leaf](../D-fms/D5-multicopter-leaf.md) *(未启动,Wave 9)* | ⇢ | §4.4.3.3 `wp_array[]` 长度;§4.4.6.3 wp / home 字段语义 |
| [D6 FMS↔Controller 接口约定](../D-fms/D6-fms-controller-interface.md) *(未启动,Wave 8)* | ↔(co-seal with E1)| §4.4.6.2 `cmd_mask`(passthrough);§4.7 cmd_mask 守卫 stub;A3 给 bit-name 工作名,D6 锁位号 |
| [E1 Controller 功能设计](../E-controller/E1-controller-functional.md) *(未启动,Wave 8)* | ↔(co-seal with D6)| §4.5 全节;§4.4.6.1 setpoint 字段(Controller 消费);§4.7 cmd_mask 守卫;§4.8 INS 消费侧依赖 |
| [E2 Controller 结构设计](../E-controller/E2-controller-structural.md) *(未启动,Wave 7)* | ⇢ | §4.5 全节 |
| [E3 Controller 各环算法](../E-controller/E3-loops-algorithm.md) *(未启动,Wave 9)* | ⇢ | §4.5 各环路 dependency |
| [E4 Controller 多旋翼 leaf](../E-controller/E4-multicopter-leaf.md) *(未启动,Wave 9)* | ⇢ | §4.3.2 / §4.5.5 `motor_cmd[]` / `actuator_cmd[]` 长度与单位 |
| [F1 ins_stub 功能设计](../F-ins-contract/F1-ins-stub-functional.md) *(未启动,Wave 6)* | ⇢ | §4.3.7 各传感器 bus 字段表(ins_stub 输入);§4.4.4 `INS_Out_Bus` 字段表(ins_stub 输出);§4.8 消费侧矩阵 |
| [F2 ins_stub 结构设计](../F-ins-contract/F2-ins-stub-structural.md) *(未启动,Wave 7)* | ⇢ | §4.3.5 `Plant_States_Bus → INS_Out_Bus` 字段映射来源 |
| [F3 INS_Out_Bus 消费规则](../F-ins-contract/F3-consumption-rules.md) *(未启动,Wave 9)* | ⇢ | §4.4.4 + §4.5.3 + §4.8 (依赖矩阵 + gate 位)|
| [G1 MIL 顶层结构设计](../G-harness/G1-mil-toplevel.md) *(未启动,Wave 10)* | ⇢ | §4.3.4 `States_Init_Bus` 注入接口;§4.3.3 `Environment_Info_Bus` 注入接口;§4.4.5 + §4.10 `Control_Out_Bus` 回送变体 |
| [G3 日志与可观测性](../G-harness/G3-logging.md) *(未启动,Wave 10)* | ⇢ | §4.9 字段方向总表 → logsout 至少包含每个 bus 的代表性字段;`timestamp` 字段(per A7) |
| [G4 Pilot_Cmd 注入设计](../G-harness/G4-pilot-injection.md) *(未启动,Wave 10)* | ⇢ | §4.4.2 `Pilot_Cmd_Bus` 字段;G4 注入序列必须覆盖该字段集 |
| [I4 Codegen 配置脚本设计](../I-tooling/I4-codegen-config.md) *(未启动,Wave 12)* | ⇢ | §4.x 各 bus 字段集 → ert.tlc 配置中 inport / outport 表;variant-conditional 字段 → "Single design" 模式裁剪 |
| [B4 契约 diff 策略设计](../B-contracts/B4-contract-diff.md) *(未启动,Wave 5)* | (transitively via B1) | §4.6 passthrough 集中表 → diff 报告中区分 generated vs passthrough 行 |

间接影响(经 B1 / B2 / B3):

- 全部 C / D / E / F 区下游;A3 是 Wave 3 的关键路径项,B 区(Wave 4)启动前必须冻结。

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-07 | A3 author | 初稿 |
| 2026-05-07 | orchestrator | reviewer verdict=pass(7/7 criteria 满足);frontmatter 升 reviewed;INDEX 已登记。commit hash 占位仍为 `<pending hash>`,待 firmware 仓挂载后由后续修订补齐 |

## Self-check

- [x] frontmatter 完整,字段值合法
- [x] 上游文档全部存在且 status ≥ reviewed(架构 v1 已发布;A1 / A2 / A4 / A5 / A6 / A7 / A8 全部 status=reviewed,2026-05-05)
- [x] 退出条件逐条复核完成,每条均给出依据(§6 表 7 项 + A1 缺口逐条闭合表)
- [x] 引用路径全部可点击访问
- [x] 不存在 RULES §5 禁则中的内容(无 .slx 截图、无可执行 .m;§4.1 ASCII 图为文本拓扑,非截图;不复述 firmware 实现细节;不重定义 byte 布局,字段类别 / 单位 / 坐标系是 field-level 语义而非 byte-level schema;FMT-Firmware 引用以路径占位 + `<pending hash>` per A6 模式)
- [x] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记(2026-05-07,orchestrator;commit hash 仍为 `<pending hash>` 占位,与 A6/A7 一致,待 firmware 仓挂载后由后续修订补齐)
- [x] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径(文件相对路径已在 §3 给出 — `FMT-Firmware/src/model/{plant,fms,control}/<vehicle>/lib/{Plant,FMS,Controller}_types.h` + `FMT-Firmware/src/model/ins/lib/INS_types.h`;commit hash 占位 `<pending hash>` 与 A6/A7 同期补齐;本文件不创建镜像产物,镜像产物由 [B1](../B-contracts/B1-bus-inventory.md) / [B5](../B-contracts/B5-ins-bus-mirror.md) 落地)
- [x] 下游影响已沿关系图识别完毕(§7 含 B1 / B2 / B3 直接 + C/D/E/F/G/I 间接,共 22 个下游)
- [x] 文档不超出本工作项范围(无越权设计:byte 布局留 B1;enum 数值留 B2;参数默认值留 B3;cmd_mask 位号留 D6/E1;FMS 启用 `Control_Out_Bus` 功能裁决留 D1;INS fallback 留 F3;harness 拓扑留 G1;variant codegen 留 I4)
