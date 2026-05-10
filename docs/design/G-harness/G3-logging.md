---
work_item: G3
title: 日志与可观测性设计
upstream: ["架构 v1", "A2", "A4", "A7", "B1", "B5", "F3", "G1", "G2"]
contract_impact: no
status: reviewed
authored_at: 2026-05-10
last_reviewed_at: 2026-05-10
reviewer_verdict: pass
---

# G3 日志与可观测性设计

## 1. 目的

为 FMT-Model-2025b MIL 仿真 harness 的 `logsout` 信号集、命名约定、采样时戳规则、单位 metadata、可视化目录与 manifest 更新规则定稿,使每次仿真运行产出**可重复消费、可回归比对、可审计追溯**的日志产物,服务于 H 区(验证)与 I5(sim runner)。本文件**只设计日志契约**,不重定义 bus/enum/参数(B 区已拥有)、不重设计速率调度(G2 已拥有)、不重设计顶层拓扑(G1 已拥有);只在三者的接缝处规定日志记录什么、怎么命名、怎么打戳、怎么写 manifest。

## 2. 范围

**在范围:**

- `logsout` **最小信号集**枚举:架构 v1 §13 各模块(Plant / Controller / FMS / INS-via-stub)的代表性信号 + INS validity(`INS_Status` + `INS_Flag` 全部位)+ `cmd_mask`(逐位)+ FMS mode/state/ext_state/failsafe_state + Controller 内部参考与 saturation flags(per E1 §4.1)+ 真值 vs 估计的对照(Plant_States_Bus vs INS_Out_Bus)
- `logsout` 信号的**命名约定**(bus 字段引用、内部 debug 信号、saturation flag 命名;ASCII)
- 每条信号的**采样时戳规则**(per A7 ms 时戳 + per A4 各模块 nominal cadence)
- `logsout` 信号的**单位 metadata** 表达方式(Simulink Signal property `DocUnits` / 数据字典)与 firmware-legacy 例外条款
- `model/harness/plots/` **可视化目录** 标准布局与每个绘图脚本的输入信号集
- 每次仿真运行的 `manifest.json` **更新规则**(字段、来源、写入时机)
- **Channel ID 稳定性**约定(`logsout` 标识在 H3 回归基线之间可比)
- **存储 / 文件格式**(`logsout.mat` Dataset 形态 + per-run 目录约定)
- 与 G1 / G2 / I5 / H1 / H2 / H3 的接缝点(由各下游消费 `logsout` + `manifest.json`)

**不在范围(由其他工作项处理):**

- bus 字段 schema(类型、宽度、offset)与 enum 数值定义 — 由 [B1](../B-contracts/B1-bus-inventory.md) / [B2](../B-contracts/B2-enum-inventory.md) 处理
- 参数 PARAM struct 字段定义 — 由 [B3](../B-contracts/B3-parameter-schema.md) 处理
- Rate transition 块设计与 sample-time 颜色检查清单 — 由 [G2](G2-rate-scheduling.md) 处理
- 顶层 `.slx` 子系统拓扑、引用模型策略、变体接入点 — 由 [G1](G1-mil-toplevel.md) 处理
- Pilot_Cmd 注入时间序列格式 — 由 [G4](G4-pilot-injection.md) 处理
- 验证场景目录与量化阈值 — 由 [H1](../H-verification/H1-scenario-catalog.md) / [H2](../H-verification/H2-metrics.md) 处理
- 回归基线版本化与对比脚本 — 由 [H3](../H-verification/H3-regression-baseline.md) 处理(本文件仅承诺 `logsout` 输出可被 H3 比对)
- `manifest.json` 的实际写入脚本 / 单/批/回归三种 CLI — 由 [I5](../I-tooling/I5-sim-runner.md) 处理(本文件仅锁字段集)
- `INS_Out_Bus` mirror 头文件与 `firmware_commit_sha` 取值 — 由 [B5](../B-contracts/B5-ins-bus-mirror.md) 处理(本文件引用 manifest 字段名)

## 3. 依赖

**KNOWN-LOOSE 注:** [G1](G1-mil-toplevel.md) 与 [G2](G2-rate-scheduling.md) 与本文件同 Wave 10,按 orchestrator wave plan 三者并行起草;依据 [01-design-relationships §4.7](../01-design-relationships.md) 的 `G1, G2 → G3` 强前置关系,本文件**不锁定** G1 的具体顶层拓扑选型与 G2 的具体 sample-time 颜色清单,只在接缝处给出"由下游决定"的占位接缝。本文件 §3 表中以 `(co-seal sibling, Wave 10)` 标注 G1/G2 的引用方式,符合 [RULES §6 self-check 第 2 项](../RULES.md) 的同 batch 兄弟例外条款。

| 上游 | 引用位置 | 用途 |
|---|---|---|
| [架构 v1 §13 (Timing)](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | §4.1 / §4.3 | 4 模块周期(Plant 1 ms / Controller 5 ms / INS 10 ms / FMS 20 ms);每模块至少 1 信号入 logsout 的最小覆盖来源 |
| [架构 v1 §14 (Parameter/Bus/Enum management)](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | §4.4 | 单位 metadata 集中管理;契约 bus 与内部 bus 区隔;参数命名 |
| [架构 v1 §17 risk 5 (timing fidelity in MIL vs SIH)](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | §4.6 | manifest.json 写入 `firmware_commit_sha`,使 MIL 跑可在未来 SIH 重跑时被锚定 |
| [A2 命名与目录约定](../A-architecture/A2-naming-conventions.md) §4.1 R-1.1 ASCII / R-1.2 大小写 / §4.5 Bus 字段命名 / §4.10 单位/坐标系/角度 | §4.2 / §4.4 | 命名风格基线;bus 字段名直接引用;单位后缀清单 |
| [A4 跨速率边界](../A-architecture/A4-rate-boundaries.md) §4.1 / §4.4 | §4.3 | 每信号采样时戳 cadence 来源 |
| [A6 Init/Reset 状态契约](../A-architecture/A6-init-reset-contract.md) §4.4 | §4.6 | reset 边界 manifest 重置语义;reset 后 logsout 段间断说明 |
| [A7 时间与时间戳约定](../A-architecture/A7-time-conventions.md) §4.1 / §4.3 / §4.5 | §4.3 | timestamp ms 单位;wrap 在 logsout 中的可视化承担 |
| [B1 Bus 清单与 schema](../B-contracts/B1-bus-inventory.md) §4.3.1 / §4.4.1 / §4.5.1 / §4.5.2 / §4.7 / §4.8 | §4.1 | 所有 contract bus 字段名(本文件以"`<Bus>.<field>`"引用,不复述 schema) |
| [B5 INS_Out_Bus 镜像同步流程](../B-contracts/B5-ins-bus-mirror.md) §4.3 / §4.5 | §4.6 | manifest `firmware_commit_sha` + `ins_bus_schema_version` 字段语义来源(B5 §4.5 / §4.7 已为 G3 manifest 占位) |
| [F3 INS_Out_Bus 消费规则](../F-ins-contract/F3-consumption-rules.md) §4.1 / §4.6.12 | §4.1 | INS validity 18 字段消费矩阵中 "harness / log" 列已 commit "必记";F3 §4.6.12 "harness / G3 logsout" 节直接锚定本文件 |
| [G1 MIL 顶层结构](G1-mil-toplevel.md) *(co-seal sibling, Wave 10)* | §4.8 / §4.9 | 顶层 .slx 提供 logsout outport;具体子系统拓扑 G1 决定 |
| [G2 多速率调度](G2-rate-scheduling.md) *(co-seal sibling, Wave 10)* | §4.3 / §4.9 | 每条 logsout 信号的 cadence 由其产出端 sample time 决定,G2 锁定具体 transitions |

`<pending hash>`:本文件不直接引用 firmware 头文件;manifest 中的 `firmware_commit_sha` 字段语义沿用 [B5](../B-contracts/B5-ins-bus-mirror.md) 已锁约定(40-char SHA1 正则),本文件不重述。

## 4. 设计内容

### 4.1 logsout 最小信号集

> **退出条件(00-design-plan §4.G G3 行)第 1 项**:logsout 最小信号集至少包含架构 v1 §13 各模块的代表性信号 + INS validity + cmd_mask + mode/state。本节按"产出方"维度逐模块枚举,每条注明信号来源 bus / 字段、cadence、单位、用途。**所有字段名以 [B1](../B-contracts/B1-bus-inventory.md) 为权威**,本表只引用名,不复述 schema。

#### 4.1.1 Plant 真值(period 1 ms,per 架构 v1 §13)

来源:Plant 输出 `Plant_States_Bus`(B1 §4.5.1)+ `Extended_States_Bus`(B1 §4.5.2)。这些是**真值**,与 INS 估计在 §4.1.3 形成 truth-vs-estimate 对照。

| logsout 通道 | 来源 bus.field | 单位 / 坐标系 | 用途 |
|---|---|---|---|
| `Plant_States_Bus.pos_ned_m` | B1 §4.5.1 行 1(`single[3]`) | m / NED | 真值位置;truth-vs-INS 对照 |
| `Plant_States_Bus.vel_ned_mps` | B1 §4.5.1 行 2 | mps / NED | 真值速度 |
| `Plant_States_Bus.acc_ned_mps2` | B1 §4.5.1 行 3 | mps2 / NED | 真值加速度(NED) |
| `Plant_States_Bus.acc_b_mps2` | B1 §4.5.1 行 4 | mps2 / body | 真值 specific force |
| `Plant_States_Bus.quat_ned_to_b` | B1 §4.5.1 行 5(`single[4]`) | dimless / NED→body | 真值姿态四元数 |
| `Plant_States_Bus.euler_ned_to_b_rad` | B1 §4.5.1 行 6 | rad / NED→body (ZYX) | 真值欧拉(可视化用) |
| `Plant_States_Bus.ang_rate_b_radps` | B1 §4.5.1 行 7 | radps / body | 真值角速度 |
| `Plant_States_Bus.motor_rpm` | B1 §4.5.1 行 11(`single[4]`) | rpm | 电机转速(quad-X 4 元素;leaf-specific 长度 per C3) |
| `Plant_States_Bus.on_ground` | B1 §4.5.1 行 12 | dimless (bool) | 接地标志 |
| `Plant_States_Bus.timestamp` | B1 §4.5.1 行 13 | ms (uint32, A7 §4.1) | Plant 端时戳 |
| `Extended_States_Bus.airspeed_mps` | B1 §4.5.2 行 1 | mps / body forward | 真空速(多旋翼 Phase 2 通常 0,但仍记) |
| `Extended_States_Bus.dynamic_pressure_pa` | B1 §4.5.2 行 4 | pa | 动压 |
| `Extended_States_Bus.disturbance_force_b_n` | B1 §4.5.2 行 6(`single[3]`) | n / body | 外扰力(故障注入 H4 验证用) |
| `Extended_States_Bus.disturbance_torque_b_nm` | B1 §4.5.2 行 7 | nm / body | 外扰力矩 |

#### 4.1.2 Plant 模拟传感器输出(period per C1 / B1 §4.8)

| logsout 通道 | 来源 bus.field | 单位 / 坐标系 | cadence | 用途 |
|---|---|---|---|---|
| `IMU_Bus.gyr_b_radps` | B1 §4.8.1 行 1 | radps / body | typical 1 ms (per B1 §4.8.1) | IMU 陀螺(含噪声/偏置) |
| `IMU_Bus.acc_b_mps2` | B1 §4.8.1 行 2 | mps2 / body | 1 ms | IMU 加速度(specific force) |
| `IMU_Bus.valid` | B1 §4.8.1 行 4 | dimless (bool) | 1 ms | sensor health(故障注入观测) |
| `IMU_Bus.timestamp` | B1 §4.8.1 行 5 | ms | 1 ms | sensor 端时戳 |
| `MAG_Bus.mag_b_gauss` | B1 §4.8.2 行 1 | gauss / body (B4 to verify gauss vs nT;G3 跟 B1 现状) | typical 10 ms | 磁场观测 |
| `MAG_Bus.valid` | B1 §4.8.2 行 2 | dimless (bool) | 10 ms | sensor health |
| `Barometer_Bus.pressure_pa` | B1 §4.8.3 行 1 | pa | typical 10 ms | 静压 |
| `Barometer_Bus.altitude_m` | B1 §4.8.3 行 3 | m | 10 ms | 气压高度 |
| `Barometer_Bus.valid` | B1 §4.8.3 行 4 | dimless (bool) | 10 ms | sensor health |
| `GPS_uBlox_Bus.lat_deg` | B1 §4.8.4 行 1 | deg | typical 100–200 ms | GPS 经纬度 |
| `GPS_uBlox_Bus.lon_deg` | B1 §4.8.4 行 2 | deg | 100–200 ms | |
| `GPS_uBlox_Bus.alt_m` | B1 §4.8.4 行 3 | m | 100–200 ms | |
| `GPS_uBlox_Bus.vel_ned_mps` | B1 §4.8.4 行 4 | mps / NED | 100–200 ms | GPS NED 速度 |
| `GPS_uBlox_Bus.num_sat` | B1 §4.8.4 行 5 | dimless | 100–200 ms | 卫星数 |
| `GPS_uBlox_Bus.fix_type` | B1 §4.8.4 行 6 (`UbloxFixType`) | enum / dimless | 100–200 ms | GPS 定位类型 |
| `GPS_uBlox_Bus.valid` | B1 §4.8.4 行 9 | dimless (bool) | 100–200 ms | sensor health |
| `AirSpeed_Bus.airspeed_mps` | B1 §4.8.5 行 1 | mps / body forward | 10–20 ms | 真空速(多旋翼 Phase 2 默认 valid=false) |
| `AirSpeed_Bus.valid` | B1 §4.8.5 行 3 | dimless (bool) | 10–20 ms | |

#### 4.1.3 INS_Out_Bus 全 18 字段(period 10 ms,per 架构 v1 §13;F3 §4.1 矩阵 "harness / log" 列已声明全字段必记)

来源:`INS_Out_Bus`(B1 §4.3.1)。在 MIL 中由 [F2 ins_stub](../F-ins-contract/F2-ins-stub-structural.md) 产出,在 SIH/SIL 中由 firmware INS 产出;本文件接收语义视同。**INS_Status 与 INS_Flag 的逐位展开**满足"INS validity"覆盖要求。

| logsout 通道 | 来源 bus.field | 单位 / 坐标系 | 用途 |
|---|---|---|---|
| `INS_Out_Bus.position_lla` | B1 §4.3.1 行 1(`double[3]` lat_deg/lon_deg/alt_m) | mixed (deg, deg, m) | INS LLA 输出 |
| `INS_Out_Bus.position_ned_m` | B1 §4.3.1 行 2 | m / NED | INS NED 位置(估计) |
| `INS_Out_Bus.velocity_ned_mps` | B1 §4.3.1 行 3 | mps / NED | INS NED 速度(估计) |
| `INS_Out_Bus.quat_ned_to_b` | B1 §4.3.1 行 4 | dimless / NED→body | INS 姿态四元数(估计) |
| `INS_Out_Bus.euler_ned_to_b_rad` | B1 §4.3.1 行 5 | rad / NED→body (ZYX) | INS 欧拉(估计) |
| `INS_Out_Bus.ang_rate_b_radps` | B1 §4.3.1 行 6 | radps / body | INS 角速度(filtered) |
| `INS_Out_Bus.acc_b_mps2` | B1 §4.3.1 行 7 | mps2 / body | INS specific force(filtered) |
| `INS_Out_Bus.INS_Status` | B1 §4.3.1 行 8 + 子 schema §4.6.5 | uint32 bitfield / dimless | INS 总状态(原始 32-bit) |
| `INS_Out_Bus.INS_Status.bit0_ready` | B2 InsStatus 位 alias(B1 §4.6.5) | dimless (bool) | INS 总体就绪;FMS arm gate |
| `INS_Out_Bus.INS_Flag` | B1 §4.3.1 行 9 + 子 schema §4.6.6 | uint32 bitfield / dimless | INS 健康位(原始 32-bit) |
| `INS_Out_Bus.INS_Flag.position_valid` | B1 §4.6.6 bit 0 | dimless (bool) | NED 位置有效 |
| `INS_Out_Bus.INS_Flag.velocity_valid` | B1 §4.6.6 bit 1 | dimless (bool) | NED 速度有效 |
| `INS_Out_Bus.INS_Flag.attitude_valid` | B1 §4.6.6 bit 2 | dimless (bool) | quat/euler 有效;**critical** |
| `INS_Out_Bus.INS_Flag.mag_valid` | B1 §4.6.6 bit 3 | dimless (bool) | 磁场观测有效 |
| `INS_Out_Bus.INS_Flag.gps_valid` | B1 §4.6.6 bit 4 | dimless (bool) | GPS 观测有效 |
| `INS_Out_Bus.INS_Flag.heading_valid` | F3 §4.1 行 / B2 InsFlag(B1 §4.6.6 占位) | dimless (bool) | yaw 角参考有效 |
| `INS_Out_Bus.INS_Flag.baro_valid` | F3 §4.1 行 / B2 InsFlag | dimless (bool) | 气压观测有效 |
| `INS_Out_Bus.INS_Flag.airspeed_valid` | F3 §4.1 行 / B2 InsFlag | dimless (bool) | 空速有效(多旋翼 Phase 2 reserved=0) |
| `INS_Out_Bus.timestamp` | B1 §4.3.1 行 10 | ms (uint32, A7 §4.1) | INS 端时戳;harness staleness oracle |

> **位扩展约定:** B1 §4.6.5 / §4.6.6 锁位 `INS_Status` / `INS_Flag` 的占位声明,具体位号由 B2 锁数值。本表所列 9 位是 F3 §4.1 已 commit 的位集(F3 §4.1 矩阵直接出现的 9 个 `INS_Flag.*`/`INS_Status.*` 行)。若 B2 后续扩位,本表通过 §4.7 channel-ID 稳定规则在末尾追加(不改变前 18 通道的 ID)。

#### 4.1.4 FMS_Out_Bus(period 20 ms,per 架构 v1 §13)

来源:`FMS_Out_Bus`(B1 §4.7.1)。**关键覆盖:`cmd_mask` 全 8 位逐位、`mode` / `ctrl_mode` / `state` / `ext_state` / `failsafe_state` / `error_code` 全部记录**,满足退出条件 "cmd_mask + mode/state" 项。

| logsout 通道 | 来源 bus.field | 单位 / 坐标系 | 用途 |
|---|---|---|---|
| `FMS_Out_Bus.pos_cmd_ned_m` | B1 §4.7.1 行 1 | m / NED | 位置命令(gated by `MASK_BIT_POSITION_LOOP`) |
| `FMS_Out_Bus.vel_cmd_ned_mps` | B1 §4.7.1 行 2 | mps / NED | 速度命令 |
| `FMS_Out_Bus.acc_cmd_ned_mps2` | B1 §4.7.1 行 3 | mps2 / NED | 加速度命令 |
| `FMS_Out_Bus.att_cmd_quat` | B1 §4.7.1 行 4 | dimless / NED→body | 姿态四元数命令 |
| `FMS_Out_Bus.ang_rate_cmd_b_radps` | B1 §4.7.1 行 6 | radps / body | 角速度命令 |
| `FMS_Out_Bus.yaw_cmd_rad` | B1 §4.7.1 行 7 | rad | yaw 角命令 |
| `FMS_Out_Bus.yaw_rate_cmd_radps` | B1 §4.7.1 行 8 | radps | yaw rate 命令 |
| `FMS_Out_Bus.throttle_cmd` | B1 §4.7.1 行 9 | dimless 0..1 (`_n01`) | throttle |
| `FMS_Out_Bus.cmd_mask` | B1 §4.7.1 行 14(uint32 bitfield) | uint32 bitfield / dimless | 原始 32-bit |
| `FMS_Out_Bus.cmd_mask.MASK_BIT_POSITION_LOOP` | D6 §4.1 / B2 cmd_mask 位 alias | dimless (bool) | 位 0(per D6 §4.1) |
| `FMS_Out_Bus.cmd_mask.MASK_BIT_VELOCITY_LOOP` | D6 §4.1 | dimless (bool) | 位 1 |
| `FMS_Out_Bus.cmd_mask.MASK_BIT_ACCELERATION_LOOP` | D6 §4.1 | dimless (bool) | 位 2 |
| `FMS_Out_Bus.cmd_mask.MASK_BIT_ATTITUDE_LOOP` | D6 §4.1 | dimless (bool) | 位 3 |
| `FMS_Out_Bus.cmd_mask.MASK_BIT_RATE_LOOP` | D6 §4.1 | dimless (bool) | 位 4 |
| `FMS_Out_Bus.cmd_mask.MASK_BIT_YAW_LOOP` | D6 §4.1 | dimless (bool) | 位 5 |
| `FMS_Out_Bus.cmd_mask.MASK_BIT_YAW_RATE_LOOP` | D6 §4.1 | dimless (bool) | 位 6 |
| `FMS_Out_Bus.cmd_mask.MASK_BIT_THROTTLE_PASSTHROUGH` | D6 §4.1 | dimless (bool) | 位 7 |
| `FMS_Out_Bus.ctrl_mode` | B1 §4.7.1 行 15(`CtrlMode` enum) | enum / dimless | 控制模式 |
| `FMS_Out_Bus.mode` | B1 §4.7.1 行 16(`FlightMode` / `PilotMode` 系列) | enum / dimless | 飞行模式;mode |
| `FMS_Out_Bus.status` | B1 §4.7.1 行 17(`VehicleStatus` enum) | enum / dimless | 总体 status |
| `FMS_Out_Bus.state` | B1 §4.7.1 行 18(`VehicleState` enum) | enum / dimless | state(D3 stateflow 输出) |
| `FMS_Out_Bus.ext_state` | B1 §4.7.1 行 19(`VehicleExtState` enum) | enum / dimless | ext_state |
| `FMS_Out_Bus.reset` | B1 §4.7.1 行 20(optional) | dimless (bool) | reset 边沿(若 firmware 含) |
| `FMS_Out_Bus.error_code` | B1 §4.7.1 行 28(`ErrorCode` enum) | enum / dimless | 错误码 |
| `FMS_Out_Bus.failsafe_state` | B1 §4.7.1 行 29(`FailsafeState` enum,optional) | enum / dimless | failsafe 子模式 |
| `FMS_Out_Bus.timestamp` | B1 §4.7.1 行 33 | ms | FMS 端时戳 |

#### 4.1.5 Control_Out_Bus + Controller 内部信号(period 5 ms,per 架构 v1 §13)

来源:`Control_Out_Bus`(B1 §4.4.1)+ Controller 内部 wire(per [E1 §4.1](../E-controller/E1-controller-functional.md) L-01..L-05 输出 + saturation latch)。

**契约 bus 字段:**

| logsout 通道 | 来源 | 单位 | 用途 |
|---|---|---|---|
| `Control_Out_Bus.motor_cmd` | B1 §4.4.1 行 1(`single[4]`) | dimless 0..1 (`_n01`) 或 us(B4 to verify;G3 跟 B1) | 电机命令(quad-X 4 元素) |
| `Control_Out_Bus.actuator_cmd` | B1 §4.4.1 行 2(`single[16]`) | mixed | 舵面 passthrough(多旋翼 Phase 2 reserved 0) |
| `Control_Out_Bus.throttle_cmd` | B1 §4.4.1 行 3 | dimless 0..1 | 总推力 echo |
| `Control_Out_Bus.cmd_mask` | B1 §4.4.1 行 4 | uint32 bitfield | passthrough echo from FMS |
| `Control_Out_Bus.ctrl_mode` | B1 §4.4.1 行 5 | enum | passthrough echo |
| `Control_Out_Bus.timestamp` | B1 §4.4.1 行 7 | ms | Controller 端时戳 |

**Controller 内部 debug 信号**(per [E1 §4.1](../E-controller/E1-controller-functional.md) 各 L 子环;**Phase 2 默认保留作 debug 通道**,per E1 行 443):

| logsout 通道(命名规则 §4.2.2) | 产出位置(E1 §4.1) | 单位 / 坐标系 | 用途 |
|---|---|---|---|
| `ctrl__vel_sp_inner_ned_mps` | E1 §4.1.2 L-01 输出(行 186) | mps / NED | L-01 派生的速度参考 |
| `ctrl__ang_rate_sp_inner_b_radps` | E1 §4.1.2 L-03 输出(行 263) | radps / body | L-03 派生的角速度参考 |
| `ctrl__thrust_cmd_b_n` | E1 §4.1.2 L-02 输出 → 进 L-05(若 B1 锁字段, otherwise 内部) | n / body | 体系推力向量(进 mixer) |
| `ctrl_sat_motor` | E1 §4.1.2 行 300 / 行 333(c)`Controller_AntiWindup_Coordinator`广播 saturation flag | dimless (bool[]) | L-05 motor_cmd 触限位 |
| `ctrl_sat_rate_roll` / `ctrl_sat_rate_pitch` / `ctrl_sat_rate_yaw` | E1 §4.1.2 L-04 each-axis saturation latch | dimless (bool) | 角速度环 anti-windup engage 标志 |
| `ctrl_sat_vel_xy` / `ctrl_sat_vel_z` | E1 §4.1.2 L-02 saturation latch | dimless (bool) | 速度环 anti-windup engage 标志 |
| `ctrl_sat_pos_xy` / `ctrl_sat_pos_z` | E1 §4.1.2 L-01 saturation latch | dimless (bool) | 位置环 anti-windup engage 标志 |

**采纳依据:** [E1 §4.1 (E1 §4.6 echo)](../E-controller/E1-controller-functional.md) 行 542 已 commit Controller 输出契约 echo `motor_cmd` / `thrust_cmd` / `actuator_cmd passthrough` / `saturation flags` / `debug` / `cmd_mask echo` / `ctrl_mode echo` / `throttle_cmd echo` / `reset echo` / `timestamp` 进 logsout;本节是 G3 侧的镜像确认。

#### 4.1.6 Pilot_Cmd / GCS_Cmd / Auto_Cmd / Mission_Data Bus(harness 输入,period 20 ms 与 FMS 对齐 per A4 RB-06)

来源:[B1](../B-contracts/B1-bus-inventory.md) §4.2.1 / §4.2.2 / §4.2.3 / §4.2.4。最小集仅记**触发与可重现性必需字段**:

| logsout 通道 | 来源 | 用途 |
|---|---|---|
| `Pilot_Cmd_Bus.stick_*` (B1 §4.2.1) | per B1 schema | 摇杆输入,可重现性 |
| `Pilot_Cmd_Bus.mode_switch` | B1 §4.2.1 + B2 PilotMode | mode 请求 |
| `Pilot_Cmd_Bus.timestamp` | B1 §4.2.1 | harness clock echo |
| `GCS_Cmd_Bus.cmd_type` (per B1 §4.2.2) | B2 GcsCmdType | GCS 命令类别(若启用) |
| `Auto_Cmd_Bus.cmd_mask_request` | B1 §4.2.3 | auto 路径请求 mask |
| `Mission_Data_Bus.wp_current_index` | B1 §4.2.4 | 当前 wp 索引 |

> **最小集裁剪原则:** Mission_Data_Bus 全字段 wp_array 不进入最小集(数据量大,变化慢);H1 验证场景若需要 wp 完整轨迹可在场景级别扩展登记。

#### 4.1.7 覆盖检查表(对应退出条件 §4.G G3)

| 退出条件清单项 | 本节覆盖位置 |
|---|---|
| 架构 v1 §13 各模块代表性信号 — Plant (1 ms) | §4.1.1 + §4.1.2 |
| 架构 v1 §13 各模块代表性信号 — Controller (5 ms) | §4.1.5 |
| 架构 v1 §13 各模块代表性信号 — INS (10 ms,via stub) | §4.1.3 |
| 架构 v1 §13 各模块代表性信号 — FMS (20 ms) | §4.1.4 |
| INS validity | §4.1.3(`INS_Status.bit0_ready` + `INS_Flag.*` 8 位) |
| cmd_mask | §4.1.4(`MASK_BIT_*` 8 位逐位) |
| mode/state | §4.1.4(`mode` / `ctrl_mode` / `status` / `state` / `ext_state` / `failsafe_state`) |

### 4.2 命名约定

> **退出条件第 2 项**:命名约定。完全沿 [A2 §4.1 R-1.1 ASCII](../A-architecture/A2-naming-conventions.md) / R-1.2 大小写;本文件不重述,仅锁定 logsout 三类信号的额外约定。

#### 4.2.1 类别 1 — Bus 字段直接引用

**G3-N-1.1**:`logsout` 通道凡来源是 contract bus(B1 已锁的 16 个 firmware-mirrored bus + harness 输入 bus),通道名采用**`<Bus_name>.<field_name>`** 精确字符串(包括大小写),例:

- `Plant_States_Bus.pos_ned_m`(NOT `plant_states_bus.pos_ned_m`,违反 A2 R-5.1 契约 bus 名固化)
- `INS_Out_Bus.INS_Flag.attitude_valid`(嵌套子 schema 用 dot path)
- `FMS_Out_Bus.cmd_mask.MASK_BIT_POSITION_LOOP`(位 alias 用 D6 §4.1 / B2 锁定的 `MASK_BIT_*` 名)

依据:A2 R-5.1 锁契约 bus 名固化、R-5.3 字段名 `lower_snake_case` + 单位/坐标系后缀;G3 仅承诺 logsout 通道名与 B1 的 `<Bus>.<field>` 一一对应,**不再在 G3 引入别名**(避免命名漂移)。

**G3-N-1.2(位 alias):** uint32 bitfield 字段(`cmd_mask`、`INS_Status`、`INS_Flag`)在 logsout 中**同时**记录原始 uint32 通道与按位 alias 通道:

- 原始:`<Bus>.<field>`(uint32)
- 位 alias:`<Bus>.<field>.<BIT_NAME>`(boolean,值为 0/1)

依据:F3 §4.1 矩阵已声明 INS_Flag 各位"必记";D6 §4.1 锁 cmd_mask 8 位语义;按位记录便于 H1/H3 直接做"该位是否在 t 时刻为 1"的断言,无需消费侧解析 bitfield。

#### 4.2.2 类别 2 — Controller 内部 debug 信号

**G3-N-2.1**:Controller 内部信号(不在任何 contract bus 上)采用 `<module>__<purpose>` 形式,**双下划线 `__` 作为分隔符**,与 contract bus 字段的单下划线区隔。模块前缀对齐 [A2 R-5.2](../A-architecture/A2-naming-conventions.md)(`ctrl_` 内部 bus 前缀大小写:本文件 logsout 用全小写 `ctrl__`,因 logsout 是数据通道而非 Simulink Bus 类型名)。

例:

- `ctrl__vel_sp_inner_ned_mps`(L-01 输出的内部速度参考,NED,m/s)
- `ctrl__ang_rate_sp_inner_b_radps`(L-03 输出的内部角速度参考,body,rad/s)
- `ctrl__thrust_cmd_b_n`(若 B1 不锁 `thrust_cmd` 字段,记入 ctrl 内部)

**G3-N-2.2**:命名末段保持 [A2 §4.10](../A-architecture/A2-naming-conventions.md) 单位/坐标系后缀(`_mps` / `_radps` / `_n` / `_rad` / `_b` / `_ned`),便于 §4.4 单位 metadata 自动派生。

**G3-N-2.3**:仅 Controller 一例引入 `<module>__<purpose>` 命名;Plant / FMS / INS-via-stub 在 logsout 中**只记 contract bus 字段**(类别 1),**不**记内部 wire(避免与 B1 schema 解耦)。Plant / FMS 内部状态如需诊断,临时在 H1 场景中以 attaching `ToWorkspace` 标量信号调试,**不**进入最小集。

#### 4.2.3 类别 3 — Saturation flags

**G3-N-3.1**:saturation flag 通道命名 `<module>_sat_<axis_or_target>`,**单下划线** + `_sat_` 中缀。`<module>` ∈ {`ctrl`}(目前仅 Controller 产出 saturation;Plant / FMS 不产出此类信号)。`<axis_or_target>` 是被饱和的对象(`motor` / `rate_roll` / `rate_pitch` / `rate_yaw` / `vel_xy` / `vel_z` / `pos_xy` / `pos_z`)。

清单(对齐 §4.1.5):

- `ctrl_sat_motor`(L-05 mixer motor_cmd 整体触限,单 boolean 或 boolean[4])
- `ctrl_sat_rate_roll` / `ctrl_sat_rate_pitch` / `ctrl_sat_rate_yaw`(L-04 各轴 anti-windup engage)
- `ctrl_sat_vel_xy` / `ctrl_sat_vel_z`(L-02)
- `ctrl_sat_pos_xy` / `ctrl_sat_pos_z`(L-01)

依据:E1 §4.1 行 188 / 229 / 300 / 333(c) Anti-windup 已声明 saturation latch 反馈给 anti-windup;本约定让每个 latch 在 logsout 中独立可观测。

#### 4.2.4 ASCII 与禁则

**G3-N-4.1**:所有 logsout 通道名一律 ASCII;**不允许**中文、空格、连字符、unicode 标识符(沿 A2 R-1.1)。反例:`电机命令`、`motor cmd`、`motor-cmd`、`motorcmd`(无可读分隔)。

**G3-N-4.2**:通道名长度 ≤ 63 字符(MATLAB 标识符上限,A2 R-1.3);典型最长形如 `FMS_Out_Bus.cmd_mask.MASK_BIT_THROTTLE_PASSTHROUGH`(50 字符),仍在限内。

### 4.3 采样时戳规则

> **退出条件第 3 项**:采样时戳规则。沿 [A7 §4.1 ms 单位](../A-architecture/A7-time-conventions.md) + [A4 §4.1 模块 cadence](../A-architecture/A4-rate-boundaries.md);本节锁定 logsout 端的时戳来源与 wrap 处理。

#### 4.3.1 每信号 cadence

**G3-T-1.1**:每条 logsout 信号的采样 cadence 由其**产出端模块周期**决定(per 架构 v1 §13 + A4 §4.1):

| 产出方 | cadence | 涵盖 §4.1 子节 |
|---|---|---|
| Plant | 1 ms (1000 Hz) | §4.1.1 + §4.1.2 IMU(典型) |
| Controller | 5 ms (200 Hz) | §4.1.5 |
| INS(MIL = ins_stub;SIH = firmware) | 10 ms (100 Hz) | §4.1.3 |
| FMS | 20 ms (50 Hz) | §4.1.4 |
| Plant 慢传感器(Mag/Baro/AirSpeed) | 10–20 ms(per B1 §4.8.x;具体由 G2 锁) | §4.1.2 |
| Plant GPS | 100–200 ms(per B1 §4.8.4;具体由 G2 锁) | §4.1.2 |
| Pilot/GCS/Auto/Mission(harness 输入) | 20 ms(对齐 FMS, per A4 RB-06) | §4.1.6 |

**G3-T-1.2**:logsout 信号的 sample time 应**与产出端 Simulink 子系统的实际 sample time 一致**(即:Simulink Logging 配置中 sample time 选 `Same as bus port` 或 `Inherited`;不在 logsout 端再做 ZOH/上采样)。具体 sample time 颜色检查由 [G2](G2-rate-scheduling.md) 负责。

#### 4.3.2 顶层 timestamp echo

**G3-T-2.1**:harness 顶层在 logsout 中**额外**写入一个独立通道 `harness__timestamp_ms`(类型 uint32,单位 ms,per A7 §4.1),由顶层 simulation clock `t * 1000` 经 `floor` 取整产生(per A7 §5 风险条目对 G1 的指引:`floor` 行为等价 firmware `systime_now_ms()`)。

依据:A7 §4.6 三模块共享同一 timestamp;harness echo 通道用作 logsout 时间轴的 master reference,便于跨模块通道(各自 1/5/10/20 ms cadence)在可视化时投影到统一时间基准。

**G3-T-2.2**:logsout 中的每条 contract-bus 时戳字段(`Plant_States_Bus.timestamp` / `INS_Out_Bus.timestamp` / `Control_Out_Bus.timestamp` / `FMS_Out_Bus.timestamp` 等)**与**`harness__timestamp_ms` 在同一 step 内**应严格相等**(per A7 §4.6 一致性承诺第 1 条 "三模块同 step 内 timestamp 完全相同")。logsout 同时记录原始 bus 时戳与 harness echo,使 H3 可对偏差做 byte-equal 断言。

#### 4.3.3 49.71 天 wrap 处理

**G3-T-3.1**:`uint32 ms` timestamp 在 49.71 天后回卷(per [A7 §4.3](../A-architecture/A7-time-conventions.md))。logsout 端**采集**:写入原始 `uint32` 值(不展开成 `uint64` 累加器),保留 firmware 行为一致性。

**G3-T-3.2**:logsout 端**可视化**:H3 比对脚本 / G3 plot 脚本若需"绝对仿真时间",采用顺序差分累加 — `t_abs[k] = t_abs[k-1] + uint32(t[k] - t[k-1])`(per A7 §4.3.1 / §4.3.2 模运算)— 显式标注 wrap 边界。**不**在 logsout 写入端做转换(避免改变 firmware-equivalent 行为)。

**G3-T-3.3**:任何依赖时戳差的 logsout 后处理算子(staleness / dt 重建)必须使用 A7 §4.3.3 比较禁则合规的方式(`int32(t_a - t_b) > 0` 而非 `t_a > t_b`)。

### 4.4 单位 metadata

> **退出条件第 4 项**:单位 metadata。沿 [A2 §4.10](../A-architecture/A2-naming-conventions.md) 单位/坐标系/角度三组约定。

#### 4.4.1 每信号单位元数据

**G3-U-1.1**:每条 logsout 信号必须**显式标注单位**。表达手段二选一(由 [G1](G1-mil-toplevel.md) / [I1](../I-tooling/I1-init-script.md) 在实施层选定):

- **方式 A — Simulink Signal property `DocUnits`**:在 Bus Object 字段上(B1 已定义)设置 `DocUnits` 字段,值取 A2 §4.10.1 单位后缀对应的 SI 字符串(`m`、`m/s`、`m/s^2`、`rad`、`rad/s`、`N`、`N*m`、`W`、`V`、`A`、`Pa`、`K`、`Hz`、`gauss`)或显式无量纲(`1` / `dimensionless`)。
- **方式 B — Data Dictionary (SLDD) 中央登记**:在 `model/shared/dict/fmt_buses.sldd`(per A2 R-3.3)对每个 Bus Object 字段附加 `DocUnits` 元数据。

**G3-U-1.2**:对 logsout 中**非 contract bus** 的信号(§4.2.2 类别 2 Controller 内部、§4.2.3 类别 3 saturation flags、§4.3.2 `harness__timestamp_ms`),通过 Simulink Signal `Signal Properties → Documentation → DocUnits` 直接在源端 outport 设置;由命名末段后缀(`_mps` → `m/s`、`_radps` → `rad/s`、`_n` → `N`、`_b` 是坐标系后缀无单位)派生。

#### 4.4.2 单位映射表(对应 A2 §4.10.1 单位后缀 → DocUnits 字符串)

| 字段后缀(A2 §4.10.1)| 物理量 | DocUnits 字符串 |
|---|---|---|
| `_m` | 长度 | `m` |
| `_mps` | 速度 | `m/s` |
| `_mps2` | 加速度 | `m/s^2` |
| `_n` | 力 | `N` |
| `_nm` | 力矩 | `N*m` |
| `_kg` | 质量 | `kg` |
| `_kgm2` | 转动惯量 | `kg*m^2` |
| `_rad` | 角度 | `rad` |
| `_radps` | 角速度 | `rad/s` |
| `_radps2` | 角加速度 | `rad/s^2` |
| `_hz` | 频率 | `Hz` |
| `_v` | 电压 | `V` |
| `_a` | 电流 | `A` |
| `_w` | 功率 | `W` |
| `_pa` | 压力 | `Pa` |
| `_k` | 温度 | `K` |
| `_c` | 温度(摄氏) | `degC` |
| `_deg` | 角度(度) | `deg` |
| `_n01` | 归一化 0..1 | `1` |
| `_pct` | 百分比 0..100 | `%` |
| `_gauss` | 磁场 | `gauss`(B4 to verify gauss vs nT;若 firmware 实际 nT,DocUnits 改 `nT` 并保留字段名 — firmware-legacy 例外) |
| `_ms` | 时间(ms) | `ms` |
| `_us` | 时间(us) | `us` |
| `_rpm` | 转速 | `rpm`(per A2 R-10.2 显式后缀,非 SI) |

#### 4.4.3 firmware-legacy 例外条款

**G3-U-2.1**:凡 firmware 已固化字段名违反 A2 §4.10 后缀规则(per A2 E-5.1 / E-10.1 例外),logsout 通道沿用 firmware 字段名,**但 DocUnits 仍按字段语义设置**。具体已知例外(per B1):

| 字段 | 命名情况 | DocUnits |
|---|---|---|
| `*.timestamp` | 字段名无 `_ms` 后缀,但 A7 §4.1 锁单位 = ms | `ms` |
| `Plant_States_Bus.motor_rpm` | 字段名带 `_rpm` 后缀(A2 R-10.2 合规)| `rpm` |
| `MAG_Bus.mag_b_gauss` | gauss 单位待 B4 verify(可能 firmware 实际 nT)| `gauss` 或 `nT`(以 B4 验证结果为准) |

**约定:** 凡 firmware-legacy 与 A2 默认 SI 不一致(例:timestamp = ms 不是 s),**logsout 跟 firmware 现状,DocUnits 写实际单位**;不在 logsout 端做单位换算(保留 firmware-equivalent 行为)。

### 4.5 可视化目录

> **退出条件第 5 项**:可视化目录。`model/harness/plots/` 下标准绘图脚本布局。

#### 4.5.1 目录与文件命名

**G3-V-1.1**:可视化脚本目录 = `model/harness/plots/`(沿 A2 §4.2 R-2.1 仓库目录树基线;A5 § / G1 进一步锁顶层 `model/harness/` 拓扑)。脚本文件命名 `plot_<topic>.m`(per A2 R-3.6 verb_object,但 `plot` 已是动词,故 `<topic>` 是 object;此处与 A2 R-3.6 等价)。

**G3-V-1.2**:**本文件给目录与脚本清单 + 每脚本输入信号集**;脚本实际实现(`.m` 内代码)由实现阶段编写,不在设计阶段写代码(per RULES §5 禁则)。

#### 4.5.2 标准绘图脚本清单

| 脚本路径 | 输入 logsout 信号集 | 用途 |
|---|---|---|
| `model/harness/plots/plot_attitude.m` | `Plant_States_Bus.quat_ned_to_b` / `Plant_States_Bus.euler_ned_to_b_rad` / `INS_Out_Bus.quat_ned_to_b` / `INS_Out_Bus.euler_ned_to_b_rad` / `FMS_Out_Bus.att_cmd_quat` | quat / euler:真值 vs INS 估计 vs FMS 命令三对照 |
| `model/harness/plots/plot_position.m` | `Plant_States_Bus.pos_ned_m` / `INS_Out_Bus.position_ned_m` / `INS_Out_Bus.position_lla` / `FMS_Out_Bus.pos_cmd_ned_m` / `Mission_Data_Bus.wp_current_index` | NED 位置 + LLA + 命令 + wp 索引 |
| `model/harness/plots/plot_velocity.m` | `Plant_States_Bus.vel_ned_mps` / `INS_Out_Bus.velocity_ned_mps` / `FMS_Out_Bus.vel_cmd_ned_mps` / `ctrl__vel_sp_inner_ned_mps` | NED 速度:真值 / INS / FMS 命令 / Controller 派生参考 |
| `model/harness/plots/plot_modes.m` | `FMS_Out_Bus.mode` / `FMS_Out_Bus.ctrl_mode` / `FMS_Out_Bus.status` / `FMS_Out_Bus.state` / `FMS_Out_Bus.ext_state` / `FMS_Out_Bus.cmd_mask`(8 位 alias)+ harness `Pilot_Cmd_Bus.mode_switch` | mode 切换 + cmd_mask 位时序 |
| `model/harness/plots/plot_motors.m` | `Control_Out_Bus.motor_cmd` / `Plant_States_Bus.motor_rpm` / `Control_Out_Bus.throttle_cmd` / `ctrl_sat_motor` / `ctrl_sat_rate_roll` / `ctrl_sat_rate_pitch` / `ctrl_sat_rate_yaw` | 电机命令 + 实际 rpm + 各轴饱和位 |
| `model/harness/plots/plot_failsafe.m` | `FMS_Out_Bus.failsafe_state` / `FMS_Out_Bus.error_code` / `FMS_Out_Bus.status` / `FMS_Out_Bus.state` + 任何 H4 故障注入触发信号(由 H1/H4 合并设计后 cross-cite) | failsafe 子模式时序 + TR-XX 触发(per D1 §4.4.1) |
| `model/harness/plots/plot_ins.m` | `INS_Out_Bus.INS_Status` / `INS_Out_Bus.INS_Status.bit0_ready` / `INS_Out_Bus.INS_Flag`(全 8 位 alias) / `INS_Out_Bus.timestamp` | INS_Status / INS_Flag 位翻转时序 + timestamp staleness |

**G3-V-1.3**:绘图脚本以**只读消费 logsout** 形式实现 — 输入 `logsout` Dataset 对象 + `manifest.json`,输出 `.fig` Simulink figure 文件 + 可选 `.png`。不修改 logsout 数据,不写回 simulation。

**G3-V-1.4**:绘图脚本的 figure 标题需嵌入 `manifest.json` 的 `scenario` + `model_repo_commit_sha`(7-char short)+ `rng_seed`(若 NOISY 变体),便于 H3 在 baseline 比对失败时快速定位 run。

### 4.6 manifest 更新规则

> **退出条件第 6 项**:manifest 更新规则。每次仿真运行产出 `manifest.json`,字段集与写入时机如下。

#### 4.6.1 manifest.json 字段集(per-run 必填)

**G3-M-1.1**:每个 per-run 输出目录下的 `manifest.json` 必须包含以下字段(顺序无关,但字段名固化):

| 字段名 | 类型 | 取值来源 | 说明 |
|---|---|---|---|
| `scenario` | string | I5 sim runner CLI 参数 / H1 场景目录名 | 场景人类可读名(例 `mc_hover_30s` / `mc_step_yaw`) |
| `firmware_commit_sha` | string (40-char hex) | B5 mirror artifact `firmware_commit_sha` header(per B5 §4.3.4) | INS_Out_Bus 字段顺序的 firmware 来源;manifest 写当前 mirror 的 SHA1 |
| `ins_bus_schema_version` | integer | B5 §4.4 ins_bus_schema_version 计数器 | INS_Out_Bus mirror 的 schema 版本号 |
| `model_repo_commit_sha` | string (40-char hex) | run-time `git rev-parse HEAD`,由 [I5](../I-tooling/I5-sim-runner.md) 自动注入 | 模型仓 commit hash |
| `vehicle` | string | sim runner CLI / `A5` 变体选择 | 机型 + 几何标记。Phase 2 默认值 = `multicopter quad-X`(per [C3](../C-plant/C3-multicopter-leaf.md))|
| `variant_mode` | string enum | sim runner CLI / F1 ideal-vs-noisy 二选一 | `IDEAL` / `NOISY`(per [F1](../F-ins-contract/F1-ins-stub-functional.md) 二变体) |
| `rng_seed` | integer (uint64) | sim runner CLI / [F1 §4.5](../F-ins-contract/F1-ins-stub-functional.md) external seed 入口(per audit F-05) | NOISY 模式必填;IDEAL 模式可填或留 null |
| `duration_s` | float | sim runner CLI / Simulink `StopTime` | 仿真总时长(秒) |
| `exit_code` | integer enum | I5 sim runner 在 stop-time 后判定 | `0` clean / `1` warn / `2` fail。判定依据由 H1/H2 锁;G3 仅锁字段 |
| `logsout_path` | string (relative path) | I5 写入相对路径 | logsout.mat 文件相对 manifest 同目录的路径(典型 `./logsout.mat`)|

**G3-M-1.2(可选字段,Phase 2 推荐):**

| 字段名 | 类型 | 说明 |
|---|---|---|
| `start_wallclock_iso` | string (ISO 8601) | run 开始时刻 wallclock(用于 audit;非物理时间)|
| `host` | string | 跑仿真的机器 hostname / CI runner ID |
| `matlab_release` | string | 例 `R2024b`(用于 H3 baseline 兼容性溯源)|
| `param_set_hash` | string (hex) | 本次 run 实际加载的 PARAM 字段集合 hash;由 I1 / I5 计算 |

**G3-M-1.3(写入时机):**

- manifest.json 由 [I5](../I-tooling/I5-sim-runner.md) sim runner 在 simulation 启动**之前**写入 `scenario` / `firmware_commit_sha` / `ins_bus_schema_version` / `model_repo_commit_sha` / `vehicle` / `variant_mode` / `rng_seed` / `duration_s` / `start_wallclock_iso` / `host` / `matlab_release` / `param_set_hash`;
- simulation 结束**之后**追加 `exit_code` / `logsout_path`。
- 若 simulation 中途 abort(超时 / hard-fail),I5 必须把已知字段写入并设 `exit_code=2`,使 manifest 始终完整(永远不出现"无 manifest" 状态)。

**G3-M-1.4**:manifest.json **是回归基线 [H3](../H-verification/H3-regression-baseline.md) 的输入元数据**;H3 比对 logsout.mat 时同时比 `firmware_commit_sha` / `ins_bus_schema_version` / `vehicle` / `variant_mode` / `rng_seed`,五项中任一不一致即触发 H3 强制人工 review(rather than 自动 pass)。

#### 4.6.2 reset 边界与 manifest

**G3-M-2.1**:本设计阶段**不**支持单 run 内多 reset。每次 simulation reset(per [A6](../A-architecture/A6-init-reset-contract.md))视为新 run,产生新 manifest.json + 新 logsout.mat 子目录。若 H1 场景需要含 reset(例如"中途 reset → 再起飞"),由 I5 / H1 在场景描述层用**多 run 顺序**模拟,每段独立 manifest。

**G3-M-2.2**:logsout 在 reset 边界**不跨 reset**(每个 run 独立 logsout 文件)。

### 4.7 Channel ID 稳定性

> **退出条件 implicit**:logsout 信号的 channel ID 在 H3 回归基线中长期可比。

**G3-C-1.1**:每条 logsout 通道获得**稳定的 channel ID**(integer 标识,与通道名字符串 1-1 绑定)。channel ID 治理规则:

| 规则 | 说明 |
|---|---|
| **R-C-1** | 每个通道名首次入 logsout 时分配一个新 ID(下一个未用整数);ID 写入 `model/harness/logsout_registry.json`(由 I5 维护)|
| **R-C-2** | ID **永不复用**。通道删除时 ID 标记为 `retired`,但**不**从 registry 移除,也**不**重分配给新通道 |
| **R-C-3** | 通道名变更视为"删旧增新":旧 ID retired,新通道获得新 ID。**不允许"原地改名 + 保留 ID"**(此约束让 H3 baseline 比对的 ID-to-name 映射稳定可逆)|
| **R-C-4** | 通道顺序在 logsout Dataset 中**不**做语义承诺 — Dataset 是无序集合,消费方按 channel 名或 channel ID 寻址,**不**按位置寻址 |
| **R-C-5** | 新增通道**只**在 registry 末尾追加;现有通道的 ID 不动 |

依据:A2 R-x Signal Naming(命名风格冻结)+ B1 §4.7.1 字段顺序 ledger(B1 已锁 firmware bus 字段 ID 稳定);G3 把同样稳定性原则延伸到 logsout channel-name → channel-ID 映射。

**G3-C-1.2**:`logsout_registry.json` schema(由 I5 维护;G3 仅锁字段名):

```pseudo
{
  "schema_version": 1,
  "channels": [
    {"id": 1,   "name": "Plant_States_Bus.pos_ned_m",        "status": "active", "added_at": "2026-MM-DD"},
    {"id": 2,   "name": "Plant_States_Bus.vel_ned_mps",      "status": "active", "added_at": "2026-MM-DD"},
    ...
    {"id": 47,  "name": "ctrl__some_old_debug",              "status": "retired","added_at": "2026-MM-DD","retired_at":"2026-MM-DD"}
  ]
}
```

**G3-C-1.3**:[H3 回归基线](../H-verification/H3-regression-baseline.md) 在比对 logsout.mat 时**优先按 channel ID 配对**;若 baseline.mat 与 current.mat 的 channel-ID 集合不一致,H3 报"channel set drift" 警告并强制人工 review,而**不**自动 fail。

### 4.8 存储 / 格式

> **退出条件 implicit**:logsout 文件存储约定。

**G3-S-1.1**:logsout 采用 Simulink Signal Logging,Dataset 格式(`Simulink.SimulationData.Dataset`)。每个 logged signal 在 Dataset 中是一个 `Simulink.SimulationData.Signal` 元素,自带 channel name + sample time + 数据。

**G3-S-1.2**:每次 run 的输出落地结构(典型;具体由 [I5](../I-tooling/I5-sim-runner.md) 锁;**G3 forward-cite I5**):

```pseudo
model/harness/runs/
├── <scenario>_<YYYYMMDD-HHMMSS>/        # per-run directory
│   ├── manifest.json                     # §4.6 字段集
│   ├── logsout.mat                       # Dataset 对象,变量名 'logsout'
│   ├── plots/
│   │   ├── attitude.fig
│   │   ├── position.fig
│   │   ├── velocity.fig
│   │   ├── modes.fig
│   │   ├── motors.fig
│   │   ├── failsafe.fig
│   │   └── ins.fig
│   └── stdout.log                        # MATLAB console capture (optional)
```

**G3-S-1.3**:`<scenario>_<YYYYMMDD-HHMMSS>` 目录命名 placeholder;实际由 I5 锁(I5 退出条件已含此项,本文件只定义"目录内必含项")。

**G3-S-1.4**:logsout.mat 是 MATLAB v7.3 (HDF5) 格式(避免 v7 旧格式 4 GB 上限);具体实现锁定由 I5 决定。

### 4.9 Cross-references / 接缝点

| 接缝目标 | G3 提供 | 下游消费 |
|---|---|---|
| [G1](G1-mil-toplevel.md) (co-seal sibling, Wave 10) | logsout outport 集合(每模块一个,共 4 + harness) | G1 顶层 .slx 引出 logsout outports |
| [G2](G2-rate-scheduling.md) (co-seal sibling, Wave 10) | 每 logsout 通道的 cadence(从产出端继承) | G2 决定 sample-time 颜色 + rate transition,本文件只承诺"按产出端 cadence 记" |
| [I5](../I-tooling/I5-sim-runner.md) (Wave 12) | manifest.json 字段集 / logsout 输出目录约定 / channel ID registry schema | I5 实现写入逻辑 + CLI |
| [H1](../H-verification/H1-scenario-catalog.md) / [H2](../H-verification/H2-metrics.md) (Wave 11) | 最小信号集 + plot 目录 + manifest 字段 | H1 场景定义 logsout 读取规则;H2 阈值在 logsout 通道上量化 |
| [H3](../H-verification/H3-regression-baseline.md) (Wave 11) | logsout.mat + manifest.json + channel ID 稳定性 | H3 baseline diff 算法直接消费 |
| [H4](../H-verification/H4-fault-catalog.md) (Wave 11) | `Extended_States_Bus.disturbance_*` + `failsafe_state` 通道 | H4 故障注入观测信号 |
| [F3 §4.6.12](../F-ins-contract/F3-consumption-rules.md) | INS validity 18 字段全记 | F3 已 commit "harness / G3 logsout 必记" 反向锁 G3 |

## 5. 已知风险与悬而未决问题

- **风险 R-G3-1:G1 / G2 同 batch 兄弟尚未起草,本文件 §4.8 / §4.9 接缝以 placeholder 形式存在**
  - 影响:G1 / G2 / G3 三者最终评审时需协同冻结。若 G1 选择 Model Reference 而非 Subsystem 作为顶层拓扑,logsout outport 接入方式可能从"顶层 Outport" 变成"Mdl ref 输出 + harness 包装",影响 §4.8 目录约定。
  - 处置:本文件**不锁** G1 拓扑选型;§4.8 / §4.9 仅约定 logsout 输出**必含项**(目录内必有 manifest.json + logsout.mat + plots/),实际目录命名与文件落地由 I5 锁;G1 / G2 评审通过后,本文件追加变更日志即可对齐。

- **风险 R-G3-2:logsout 通道命名空间随设计演进可能爆增**
  - 影响:Phase 2 仅多旋翼 + IDEAL/NOISY,通道数 ≈ 80;Phase 5 多机型扩展后可能膨胀到 300+,registry 维护成本上升。
  - 处置:R-C-2 / R-C-3 已锁"never-reuse + 末尾追加"治理。本文件不预留多机型通道,Phase 2 closed后由 I5 / 多机型 leaf 设计再扩展。

- **风险 R-G3-3:Controller 内部 debug 信号(§4.1.5 + §4.2.2)默认进入 logsout 但 E5 性能预算可能拒收**
  - 影响:E1 §4.1 行 443 已 commit "Phase 2 默认保留作 debug 通道",但 E5(Wave 10 sibling)若发现 5 ms 周期内额外信号采集越界,需把这些通道改为 `Test_Point` 形态(运行时可关)。
  - 处置:**Open;由 [E5 性能预算设计](../E-controller/E5-performance-budget.md) 在 Wave 10 协同评审时复核**。本文件承诺:logsout 通道集**可裁剪**(以 sim runner CLI flag 启用 / 禁用),最小集仅强制 §4.1.1 / §4.1.3 / §4.1.4 / §4.1.5 contract bus 部分;§4.1.5 末尾的 ctrl 内部 debug + saturation flags 是 **Phase 2 default-on,可关**。具体开关由 I5 锁。

- **风险 R-G3-4:DocUnits 在 codegen 中可能丢失**
  - 影响:若 [I4 codegen 配置](../I-tooling/I4-codegen-config.md) 选择 ert.tlc 标准模板,Simulink Signal `DocUnits` 不会嵌入生成的 `.c` / `.h`(只在 SLDD 与 Simulink workspace 中)。但 logsout 是 MIL-only 产物,不进 codegen,不构成实际损失。
  - 处置:确认 logsout 仅 MIL 用,**不**进 codegen;本文件 §4.4 锁定的 DocUnits 仅服务可视化与 H 区消费。

- **风险 R-G3-5:firmware-legacy 单位例外(Mag gauss vs nT,timestamp ms 而非 s)需 B4 verify 后才能定值**
  - 影响:§4.4.3 例外条款依赖 B4 contract diff 在首次 export 时验证 firmware 实际单位。
  - 处置:本文件按 B1 现状(`gauss` / `ms`)登记;B4 验证后若发现 firmware 实际为 `nT` / `s`,本文件 DocUnits 表追加变更日志。

- **(open) cmd_mask 8 位位号在 B2 锁定数值前为占位**
  - 影响:`MASK_BIT_POSITION_LOOP=0` / `MASK_BIT_VELOCITY_LOOP=1` / ... 的具体位号由 D6 §4.1 + B2 cmd_mask alias 锁;若位号变化,§4.1.4 的 alias 通道名仍按位**名**而非位**号**记录(命名稳定),无影响。
  - 处置:G3 logsout 用位**名** alias(`MASK_BIT_POSITION_LOOP`)而不是位**号**(`bit0`),与位号变更解耦。

- **(open) Phase 2 多旋翼以外的机型扩展**
  - 影响:Phase 5 扩展到 fixwing/vtol/boat 后,Plant 输出 bus 字段集与传感器集会扩张;§4.1.2 / §4.1.5 需追加 leaf-specific 通道。
  - 处置:本文件 Phase 2 scope 不裁决多机型;追加在 Phase 5 leaf 设计中以变更日志方式追加(R-C-5)。

## 6. 退出条件复核

[`00-design-plan.md §4.G`](../00-design-plan.md) G3 退出条件原文:

> logsout 最小信号集(至少包含架构 v1 §13 各模块的代表性信号 + INS validity + cmd_mask + mode/state)、命名约定、采样时戳规则、单位 metadata、可视化目录、manifest 更新规则

逐条复核:

| # | 退出条件分解 | 本文档依据 | 状态 |
|---|---|---|---|
| 1 | 最小信号集:架构 v1 §13 各模块代表性信号(Plant 1 ms / Controller 5 ms / INS 10 ms / FMS 20 ms) | §4.1.1(Plant)+ §4.1.2(Plant 传感器)+ §4.1.3(INS via stub)+ §4.1.4(FMS)+ §4.1.5(Controller);§4.1.7 覆盖检查表 | 满足 |
| 2 | 最小信号集:INS validity | §4.1.3 含 `INS_Out_Bus.INS_Status.bit0_ready` + `INS_Out_Bus.INS_Flag.*` 8 位逐位 + 原始 uint32 通道 | 满足 |
| 3 | 最小信号集:cmd_mask | §4.1.4 含 `FMS_Out_Bus.cmd_mask` 原始 + `MASK_BIT_*` 8 位 alias 逐位 | 满足 |
| 4 | 最小信号集:mode/state | §4.1.4 含 `mode` / `ctrl_mode` / `status` / `state` / `ext_state` / `failsafe_state` / `error_code` | 满足 |
| 5 | 命名约定 | §4.2(三类:G3-N-1 bus 字段 / G3-N-2 内部 debug `<module>__<purpose>` / G3-N-3 saturation `<module>_sat_<axis>` / G3-N-4 ASCII + 长度) | 满足 |
| 6 | 采样时戳规则 | §4.3(G3-T-1 cadence 来自产出端 + G3-T-2 harness echo timestamp_ms + G3-T-3 wrap 处理) | 满足 |
| 7 | 单位 metadata | §4.4(G3-U-1 DocUnits 表达 + G3-U-1 单位映射表 25 项 + G3-U-2 firmware-legacy 例外条款) | 满足 |
| 8 | 可视化目录 | §4.5(`model/harness/plots/` 7 脚本清单 + 每脚本输入信号集) | 满足 |
| 9 | manifest 更新规则 | §4.6(G3-M-1 11 必填字段 + 4 可选字段 + 写入时机 + reset 边界处理 + H3 metadata 守门) | 满足 |

附:audit-fortified 退出条件(00-design-plan §4.G G3 行)所有六项均已在 §4.1 / §4.2 / §4.3 / §4.4 / §4.5 / §4.6 章节闭合。

## 7. 下游影响

按 [`01-design-relationships.md §4.7 / §4.8`](../01-design-relationships.md):

```text
G1, G2 → G3
G3 ⇢ H3
```

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [H1 验证场景目录](../H-verification/H1-scenario-catalog.md) *(Wave 11,未启动)* | ⇢ | logsout 最小信号集(§4.1)定义场景"必看"信号;plot 目录(§4.5)给场景输出建议 |
| [H2 验证指标](../H-verification/H2-metrics.md) *(Wave 11,未启动)* | ⇢ | §4.1 信号集是 H2 阈值量化的载体;`<Bus>.<field>` 命名(§4.2)是阈值表达的字段路径 |
| [H3 回归基线](../H-verification/H3-regression-baseline.md) *(Wave 11,未启动)* | ⇢ | manifest.json 字段(§4.6)是 baseline metadata;channel ID 稳定性(§4.7)使 baseline 跨版本可比;logsout.mat Dataset 格式(§4.8)是 baseline 主体 |
| [H4 故障注入](../H-verification/H4-fault-catalog.md) *(Wave 11,未启动)* | ⇢ | `Extended_States_Bus.disturbance_*`(§4.1.1)+ sensor `valid` 位(§4.1.2)+ `failsafe_state`(§4.1.4)是 H4 注入效果观测的接入点 |
| [I5 仿真运行/批量回归](../I-tooling/I5-sim-runner.md) *(Wave 12,未启动)* | ⇢ | manifest.json 字段集(§4.6.1)+ run 输出目录约定(§4.8.2)+ channel ID registry schema(§4.7) — I5 实现这些落地逻辑 |

**间接影响 / 邻近责任方:**

| 受影响方 | 影响内容 |
|---|---|
| [G1 MIL 顶层](G1-mil-toplevel.md) *(co-seal sibling, Wave 10)* | logsout outport 须从顶层每个模块子系统引出;具体接入点由 G1 锁。本文件不裁决 |
| [G2 多速率调度](G2-rate-scheduling.md) *(co-seal sibling, Wave 10)* | logsout 信号 cadence 与 G2 sample-time 颜色一致;本文件只承诺"按产出端记" |
| [G4 Pilot_Cmd 注入](G4-pilot-injection.md) *(Wave 10,未启动)* | Pilot_Cmd_Bus 字段记入 logsout(§4.1.6);G4 的 DSL 输出格式独立 |
| [E5 Controller 性能预算](../E-controller/E5-performance-budget.md) *(Wave 10,未启动)* | §4.1.5 Controller 内部 debug 通道默认 Phase 2 ON;E5 评审时若发现性能不足,可裁剪为 `Test_Point` runtime-toggleable(R-G3-3 已记) |
| [B5 INS_Out_Bus mirror](../B-contracts/B5-ins-bus-mirror.md) *(Wave 5,已 reviewed)* | manifest.json 的 `firmware_commit_sha` / `ins_bus_schema_version` 字段直接复用 B5 §4.3.4 / §4.4 锁定语义 |
| [F1](../F-ins-contract/F1-ins-stub-functional.md) / [F2](../F-ins-contract/F2-ins-stub-structural.md) ins_stub *(Wave 6/7,已 reviewed)* | §4.1.3 INS 字段 18 通道在 MIL 中由 ins_stub 产出;F1 / F2 已设计完成,本文件只消费输出 |

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-10 | fmt-design-author (G3 worker) | 初稿 |

## Self-check

- [x] frontmatter 完整,字段值合法
- [x] 上游文档全部存在且 status ≥ reviewed,或为同 co-seal batch 兄弟(已在 §3 注明 Wave 10 sibling = G1 / G2,符合 KNOWN-LOOSE 注与 RULES §6 例外条款)
- [x] 退出条件逐条复核完成,每条均给出依据(§6 表 9 行)
- [x] 引用路径全部可点击访问(架构 v1 / A2 / A4 / A6 / A7 / B1 / B5 / E1 / F3 / G1 / G2 / G4 / I5 / H1 / H2 / H3 / H4 / RULES / 00-design-plan / 01-design-relationships)
- [x] 不存在 RULES §5 禁则中的内容(无 .slx 截图、无可执行 .m 代码;`logsout_registry.json` 字段示例标 `pseudo`;无重复 firmware 实现细节;无重复 bus/enum/PARAM 字段定义;依赖均在 §3 列出;不引用 PR/branch 名)
- [N/A] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记 — **本文件 contract_impact=no**(logsout 是 MIL-only 产物,不进入 firmware 可见接口);N/A
- [N/A] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径 — 本文件不镜像 firmware 头文件;`firmware_commit_sha` 仅作为 manifest 字段名引用 B5 已有约定;N/A
- [x] 下游影响已沿关系图识别完毕(§7 直接出边 H1/H2/H3/H4/I5 + 间接 G1/G2/G4/E5/B5/F1/F2)
- [x] 文档不超出本工作项范围(无越权设计) — 不重定义 bus schema(只引用 B1)、不重设计 rate scheduling(只引用 G2)、不重设计顶层拓扑(只引用 G1);存储格式与 CLI 实现 forward-cite I5
