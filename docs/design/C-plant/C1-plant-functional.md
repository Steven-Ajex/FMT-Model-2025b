---
work_item: C1
title: Plant 功能设计
upstream: [架构 v1, A1, A2, A3, A4, A5, A6, A7, A8, B1, B2, B3]
contract_impact: no
status: draft
authored_at: 2026-05-08
last_reviewed_at:
reviewer_verdict: none
---

# C1 Plant 功能设计(Plant Functional Design)

## 1. 目的

为 FMT-Model-2025b Plant 模块定义**功能层契约**:在不触及结构分解(C2)、数值积分(C4)、机型几何参数(C3 leaf)的前提下,锁定 (a) Plant 在 Phase 2 多旋翼场景下**仿真的物理范围**(刚体 / 气动 / 推进 / 传感器 / 扰动 / 环境 / init),(b) 每个组件的**保真度等级**(1=ideal … 4=HIL-class)及其取舍依据,(c) 各组件的**功能行为**与**输入/输出契约回声**,(d) 传感器测量模型的功能层定义,(e) init/reset 行为映射到 [A6](../A-architecture/A6-init-reset-contract.md) 的 Reset 类别。本文件闭合 [00-design-plan §4.C](../00-design-plan.md) C1 退出条件:"仿真物理范围(刚体/气动/电机/传感器/扰动)与各项物理保真度等级"。

## 2. 范围

**在范围:**
- 仿真物理范围 §4.1 enumerated:刚体 6-DoF / 气动 / 推进 / 传感器 / 扰动 / 环境 / init/reset
- 每组件保真度等级 §4.2(1–4 ladder + Phase 2 默认值 + 分级理由)
- 输入契约回声 §4.3(`Control_Out_Bus` / `Environment_Info_Bus` / `States_Init_Bus`,字段级清单引用 A3 / B1)
- 输出契约回声 §4.4(`Plant_States_Bus` / `Extended_States_Bus` / 模拟传感器 buses,字段级引用 A3 / B1)
- 各组件功能行为 §4.5(输入信号、输出信号、关键方程文本/伪代码、保真度条件分支)
- 传感器测量模型功能定义 §4.6(truth source / 噪声类 / 偏置类 / cadence / validity)
- init/reset 行为(per A6)§4.7
- 扰动注入点的**功能层**定位 §4.8(注入到哪个物理量;具体结构由 C2)
- Plant 慢传感器 cadence 数值锁定(per [A4 RB-07](../A-architecture/A4-rate-boundaries.md) 委托)§4.6

**不在范围(由其他工作项处理):**
- Plant 内部子系统层次、共享 vs leaf 边界、库块清单 — 由 [C2](C2-plant-structural.md) 处理
- 多旋翼几何 / 质量惯量 / 电机模型常数 / 分配矩阵 / 数值默认 — 由 [C3](C3-multicopter-leaf.md) 处理
- 数值积分器选择 / 步长 / reset 内部行为 / 数值稳定性边界 — 由 [C4](C4-numerics.md) 处理
- 传感器噪声 σ / 偏置数值 / 风场数值 — 由 [B3 §4.3](../B-contracts/B3-parameter-schema.md) `PLANT_PARAM` 表处理(本文件仅定义噪声**模型类**)
- bus 字段级 schema(byte offset / 类型 / 顺序) — 由 [B1](../B-contracts/B1-bus-inventory.md) 拥有
- enum 数值锁定 — 由 [B2](../B-contracts/B2-enum-inventory.md) 拥有
- `ins_stub` 内部如何从 `Plant_States_Bus` 派生 `INS_Out_Bus` — 由 [F1](../F-ins-contract/F1-ins-stub-functional.md) / [F2](../F-ins-contract/F2-ins-stub-structural.md) 处理(C1 仅承诺发布 truth)
- 故障注入分类 / 接口 / 触发参数 — 由 [H4](../H-verification/H4-fault-catalog.md) 处理(C1 仅给出注入**点位**)
- 固定翼 / VTOL 特有气动子模型 — 由 [A5 Phase 5 路径](../A-architecture/A5-variant-strategy.md) 推迟
- INS 估计算法 — firmware-owned(架构 v1 §3 / §12.3)
- 多旋翼以外机型的 leaf 设计 — 00-design-plan §5 排除

## 3. 依赖

环境注:FMT-Firmware 在本设计阶段未挂载,作者无法直接读取 `plant_interface.h` / `Plant_types.h`。本文件用 `FMT-Firmware @ <pending hash>` 占位指代 firmware tip;契约级 byte-for-byte 验证延迟到 [B4](../B-contracts/B4-contract-diff.md) 阶段。

| 上游 | 引用位置 | 用途 |
|---|---|---|
| 架构 v1 §3 / §7.1 / §8.1 / §12.4 / §13 / §15.1 | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | Plant mission(simulate physics)/ Plant 模块 split / Plant boundary / Plant 6 阶段 / Plant 1 ms / MIL `ins_stub` 闭环路径 |
| 架构 v1 §17 risk 1 / risk 4 / risk 5 | 同上 | 契约 drift / `ins_stub` 现实性 / 时序 realism;C1 保真度等级与扰动注入决策需呼应 |
| [A1 §5.1.7 / §8.1 / §12.4 / §15.1 缺口](../A-architecture/A1-v1-review.md) | A1 表 | "optional 传感器准则未定" / "各阶段保真度等级未定" / "ins_stub 噪声范围" — C1 闭合 §5.1.7 / §12.4 中"保真度等级"部分;§15.1 噪声范围由 F1 闭合(C1 仅定义模型类) |
| [A2 命名 / 单位 / 坐标系约定](../A-architecture/A2-naming-conventions.md) | A2 R-5 / R-7 / R-10 | 本文件信号名 / 单位后缀 / 坐标系标注遵循 A2 |
| [A3 §4.3 Plant 模块边界](../A-architecture/A3-module-boundaries.md) | A3 §4.3.1–4.3.7 | 输入 / 输出 bus 字段级清单(C1 §4.3 / §4.4 仅引用,不重定义) |
| [A4 §4.4 RB-01 / RB-02 / RB-07](../A-architecture/A4-rate-boundaries.md) | A4 §4.4 | RB-02(Controller→Plant ZOH)/ RB-07(Plant 内部慢传感器 cadence)— A4 把"具体 cadence 值"委托给 C1,本文件 §4.6 锁定 |
| [A5 §4.2 Phase 2 多旋翼](../A-architecture/A5-variant-strategy.md) | A5 §4.2 | Phase 2 仅 multicopter;传感器配置冻结为 IMU+MAG+Baro+GPS,AirSpeed variant-conditional |
| [A6 §4.4 reset 类别](../A-architecture/A6-init-reset-contract.md) | A6 §4.4.1 / §4.4.2 / §4.5.2 / §4.6 | C1 §4.7 init/reset 行为映射 PARAM / HARDCODED / ZERO / PRESERVED 四类 |
| [A7 timestamp 约定](../A-architecture/A7-time-conventions.md) | A7 §4.1 / §4.6 | Plant 输出 bus 中 `timestamp` 字段的语义来源 |
| [A8 §4.3.1 Plant Layer B 候选块](../A-architecture/A8-shared-library-roster.md) | A8 §4.3.1 | C1 §4.5 功能描述与 A8 候选共享块(`Plant_Sensor_Synthesis_Shell` / `Rigid_Body_Dynamics_Shell` / `Disturbance_Injector_Shell` / `Plant_Reset_Hooks_Shell`)的功能边界呼应,**结构归属由 C2** |
| [B1 §4.5 / §4.8 Plant 输出 buses](../B-contracts/B1-bus-inventory.md) | B1 §4.5.1 / §4.5.2 / §4.5.3 / §4.5.4 / §4.8.1–§4.8.5 | `Plant_States_Bus` / `Extended_States_Bus` / `Environment_Info_Bus` / `States_Init_Bus` / IMU/MAG/Baro/GPS/AirSpeed bus schema 全在 B1 |
| [B2 enum](../B-contracts/B2-enum-inventory.md) | B2 | `UbloxFixType`(GPS fix_type)/ gear_state 等 enum 数值由 B2 锁定 |
| [B3 §4.3 PLANT_PARAM 34 字段](../B-contracts/B3-parameter-schema.md) | B3 §4.3.2 | C1 引用 PLANT_PARAM.01–.34;具体数值与运行时 vs 编译期分类由 B3 |

co-seal batch:无(C1 是 Wave 6 单兵作战;A 区 / B 区上游全部 reviewed)。

## 4. 设计内容

### 4.1 仿真物理范围(physics scope enumeration)— 闭合架构 v1 §12.4 与退出条件 (a)

Phase 2(多旋翼)Plant 仿真 **七**类物理 / 行为组件,覆盖架构 v1 §12.4 列出的 6 阶段加上"init / reset"行为类:

| # | 组件 | 物理 / 行为类 | 主要状态 / 输出关系 | 架构 v1 §12.4 阶段对照 |
|---|---|---|---|---|
| C1.P1 | **刚体 6-DoF**(Rigid body) | 平动 + 转动动力学;NED 系位置 / 速度 / 加速度,体系角速度 / 角加速度,姿态(quat NED→body) | 状态向量 = (`pos_ned_m`, `vel_ned_mps`, `quat_ned_to_b`, `ang_rate_b_radps`);`Plant_States_Bus` 主体 | "Vehicle Dynamics Core" + "Kinematics/Coordinate Conversion" |
| C1.P2 | **气动**(Aerodynamics) | 阻力 / 地效 / prop wash(若启用) | `disturbance_force_b_n` / `disturbance_torque_b_nm` 之一部分;不含升力面(多旋翼无固定翼面) | "Vehicle Dynamics Core"(气动子集) |
| C1.P3 | **推进 / 电机**(Propulsion) | 电机推力 + 反扭矩 by `Control_Out_Bus.motor_cmd` → 体系合力 / 合力矩 + `motor_rpm` 状态 | 电机一阶动态(由 PLANT_PARAM.10 `motor_time_const_s`)+ 推力曲线(PLANT_PARAM.11) | "Actuator/Airframe-Specific Dynamics" |
| C1.P4 | **传感器合成**(Sensor synthesis) | 由 truth state 派生 `IMU_Bus` / `MAG_Bus` / `Barometer_Bus` / `GPS_uBlox_Bus` / `AirSpeed_Bus`(后者 variant-conditional) | truth + 噪声/偏置模型 | "Sensor Synthesis" |
| C1.P5 | **扰动注入**(Disturbances) | 风 / 阵风 / 湍流 / 传感器噪声 / 电机故障(per [H4](../H-verification/H4-fault-catalog.md) forward-cite) | 消费 `Environment_Info_Bus.wind_vel_ned_mps` 等;输出附加体系合力 / 合力矩 | "Environment/Disturbance Inputs" |
| C1.P6 | **环境模型**(Environmental) | 重力 / 磁场 / 大气(密度 / 压力 / 温度) | 全部由 `Environment_Info_Bus`(per A3 §4.3.3 / B1 §4.5.3)注入,Plant 不内部建模(常量) | "Environment/Disturbance Inputs"(常量端) |
| C1.P7 | **Init / reset** | 初值注入(`States_Init_Bus`)+ reset 时回到 PARAM / HARDCODED / ZERO 稳态(per A6 §4.4.2) | 见 §4.7 | "Output Assembly" 的 reset 通路 |

注:架构 v1 §12.4 列了 6 阶段(Environment/Disturbance Inputs → Vehicle Dynamics Core → Actuator/Airframe-Specific Dynamics → Kinematics/Coordinate Conversion → Sensor Synthesis → Output Assembly);C1 把"Output Assembly"中的 reset 通路独立成 P7,以与 [A6](../A-architecture/A6-init-reset-contract.md) 显式对接。这只是**功能划分**,结构层(C2)如何把它们落到子系统块图是 C2 决定。

### 4.2 保真度等级(fidelity ladder)— 闭合退出条件 (b)

#### 4.2.1 保真度等级定义(1–4 ladder)

| Level | 名称 | 含义 | 典型代价 / 信号特征 |
|---:|---|---|---|
| **L1** | **Ideal** | 直接耦合 / 零延迟 / 无噪声 / 代数关系直出 | 最低计算成本;状态变化"无物理性",仅满足契约 bus 字段非空 |
| **L2** | **First-order** | 一阶动态 / 线性近似 / 高斯白噪声 + 固定偏置 / 单一时间常数 | 中等成本;能反映主导动力学,但忽略高阶耦合 |
| **L3** | **High-fidelity-MIL** | 多阶 / 非线性 / 颜色噪声(随机游走偏置 + 白噪声)/ 多时间常数 / 显式延迟 | 高成本;接近 SIH 行为,适合算法验证 |
| **L4** | **HIL-class** | 含传感器物理细节(saturation / 量化 / drift / 温度耦合)/ 真实带宽 / 数据包损失 / 时基抖动 | 仿真预算最高;通常仅在 SIH/HIL 阶段启用 |

任意组件可在 PARAM 配置下从 L1 升级到更高 level(由 C2 / C3 决定哪些可切换、由 PLANT_PARAM 哪些字段做 switch);Phase 2 默认值固化为下表。

#### 4.2.2 Phase 2 多旋翼默认保真度

| 组件(§4.1 ID) | Phase 2 默认 | 分级理由 | 升级路径(Phase 3+) |
|---|---:|---|---|
| **C1.P1 刚体 6-DoF** | **L4** | 闭环算法验证的"真值"基础;若刚体本身 ≤ L3,则上层 FMS / Controller 评测无意义。L4 = 完整 6-DoF Newton-Euler + 四元数姿态,无小角度近似 | 已在 L4;HIL 时可加 IMU 安装偏角等次要修正 |
| **C1.P2 气动** | **L2** | Phase 2 多旋翼以低速 / 悬停 / 慢速航点为主;气动主导项 = 平动阻力(linear + quadratic),次要项(地效 / prop wash / 旋翼-旋翼干扰)在低速下贡献小。由 PLANT_PARAM.12 / .13 提供 σ-known 阻力系数;不含 CFD-级效应 | L3:加入地效模型(高度依赖)+ 简化 prop wash;L4:推迟到 fixed-wing 阶段(气动是主导力) |
| **C1.P3 推进 / 电机** | **L2** | 一阶电机响应(`motor_time_const_s` PARAM.10)+ 多项式推力曲线(PARAM.11)足以反映 ESC + 电机 + 螺旋桨主导动态。不含电池 sag、温度漂移、ESC 死区 | L3:加电池 sag(电池电压 → 推力上限折损);L4:含 ESC 控制延迟与电流环 |
| **C1.P4 传感器合成** | **L3** | 闭环算法验证(尤其姿态 / 位置环)对 IMU 噪声 / 偏置非常敏感;若 ≤ L2 则 ins_stub / 真实 INS 行为差异被掩盖,违反架构 v1 §17 risk 4。L3 = Gaussian 噪声 + 随机游走偏置 + 显式 cadence | L4:加入 IMU saturation / 量化 / 温度漂移;延迟到 SIH 阶段 |
| **C1.P5 扰动注入** | **L1**(默认禁用 / 常量风)→ **L2**(由 H1 / H4 启用时) | Phase 2 默认场景为"零风、无故障"(per A3 §4.3.3 末尾"Phase 2 默认 H4 不启用动态扰动");H1 验证场景按需启用恒定风(L1)或一阶 Dryden 湍流(L2);故障注入(电机失效)由 H4 forward-cite | L3:Dryden / Von Karman 全谱湍流 + 阵风模型;L4:CFD-class 风场 |
| **C1.P6 环境模型** | **L1** | 全部由 `Environment_Info_Bus` 配置时常量(重力 / 磁场 / 大气 ISA),per A3 §4.3.3 + B1 §4.5.3。Phase 2 不模拟高度依赖大气、不模拟磁场地理变化、不模拟 GPS 卫星几何 | L2:高度依赖大气模型 + WMM 磁场;L3:GPS 卫星几何 + 多路径效应(推迟 SIH) |
| **C1.P7 Init / reset** | **L4**(契约级,无 ladder) | 不属于物理保真度,而是契约行为;必须 100% 符合 A6 §4.4 类别契约,无降级空间 | N/A |

**保真度等级总线**:L1 (P5 默认 + P6) / L2 (P2 + P3 + P5 启用) / L3 (P4) / **L4 (P1 + P7)**。Phase 2 整体保真度 = 限制项为 P2 / P3 = **L2-bounded**(气动与推进的一阶近似是闭环验证的现实性下限)。

#### 4.2.3 推迟到 Phase 3+ / SIH 的项

- **C1.P1 IMU 安装偏角校正**(L4+)— SIH 阶段(架构 v1 §15.3)
- **C1.P2 prop wash / 地效 高保真**(L3+)— C2 / C3 评估若 H1 验证发现近地稳定性问题再升级
- **C1.P3 电池 sag + ESC 动态**(L3+)— Phase 4 性能调优(架构 v1 §18 Phase 4)
- **C1.P4 IMU saturation / 温度漂移**(L4)— SIH
- **C1.P5 全谱湍流 / CFD 风场**(L3+)— H4 故障分类成熟后由 H1 增量启用
- **C1.P6 高度依赖大气 + WMM 磁场**(L2+)— 长航时场景(Phase 4+)

### 4.3 输入契约(input contract)— echo from A3 §4.3.2–4.3.4 + B1 §4.5.3 / §4.5.4

Plant 在 MIL 闭环中只读三条输入 bus(架构 v1 §8.1 / A3 §4.3.1)。本节**仅引用**字段级清单,不重定义。

| Bus | 来源 | 速率边界 | mandatory / optional | 字段级清单源 | Reset 类别 |
|---|---|---|---|---|---|
| `Control_Out_Bus` | Controller 5 ms | RB-02 ZOH(per A4 §4.4) | mandatory(MIL 闭环) | [A3 §4.3.2](../A-architecture/A3-module-boundaries.md#432-plant-输入control_out_bus) + [B1 §4.4.1](../B-contracts/B1-bus-inventory.md#441-control_out_bus) | per A6 §4.4.2 Control_Out_Bus 行 |
| `Environment_Info_Bus` | scenario / harness | configuration-time(non-step bus) | mandatory(为 P5 / P6 提供常量) | [A3 §4.3.3](../A-architecture/A3-module-boundaries.md#433-plant-输入environment_info_bus) + [B1 §4.5.3](../B-contracts/B1-bus-inventory.md#453-environment_info_bus) | 全字段 PARAM(per A6 §4.4.2) |
| `States_Init_Bus` | scenario / harness | configuration-time(仅 init / reset 时被读) | mandatory(P7 init 需求) | [A3 §4.3.4](../A-architecture/A3-module-boundaries.md#434-plant-输入states_init_bus) + [B1 §4.5.4](../B-contracts/B1-bus-inventory.md#454-states_init_bus) | 全字段 PARAM(per A6 §4.4.2) |

**Plant 不读取以下 bus(显式排除以防越界)**:`FMS_Out_Bus`(Plant 不消费 FMS 决策)/ `INS_Out_Bus`(Plant 不消费估计结果,Plant 是估计器的输入源)/ `Pilot_Cmd_Bus` / `GCS_Cmd_Bus` / `Auto_Cmd_Bus` / `Mission_Data_Bus`(全部 FMS-only)。

**Harness-only 输入(非契约级,仅 G1 harness 内通路)**:无。Phase 2 全部 Plant 输入都在上表。

### 4.4 输出契约(output contract)— echo from A3 §4.3.5–4.3.7 + B1 §4.5 / §4.8

Plant 的输出 bus 七条(`Plant_States_Bus` + `Extended_States_Bus` + 五条传感器 bus)。本节**仅引用**字段级清单。

| Bus | 去向(MIL) | 速率边界 | mandatory / optional / variant-conditional | 字段级清单源 | 全字段 reset 类别 |
|---|---|---|---|---|---|
| `Plant_States_Bus` | `ins_stub` 主消费;Controller 仅 harness variant | 内部至 ins_stub(同 1 ms 主循环);RB-01 仅 variant | mandatory(MIL truth) | [A3 §4.3.5](../A-architecture/A3-module-boundaries.md#435-plant-输出plant_states_bus) + [B1 §4.5.1](../B-contracts/B1-bus-inventory.md#451-plant_states_bus) | per A6 §4.4.2 Plant_States_Bus 行(刚体字段 PARAM,电机字段 ZERO 默认,timestamp ZERO,on_ground HARDCODED true) |
| `Extended_States_Bus` | harness / observer | RB-01(harness-only) | mandatory(harness 日志);多旋翼大部分字段 ZERO | [A3 §4.3.6](../A-architecture/A3-module-boundaries.md#436-plant-输出extended_states_bus) + [B1 §4.5.2](../B-contracts/B1-bus-inventory.md#452-extended_states_bus) | 全字段 ZERO(per A6 §4.4.2) |
| `IMU_Bus` | `ins_stub` | RB-07 内部 cadence ≈ 1 ms | mandatory | [A3 §4.3.7.1](../A-architecture/A3-module-boundaries.md#4371-imu_bus典型-cadence--1-ms与-plant-同步) + [B1 §4.8.1](../B-contracts/B1-bus-inventory.md#481-imu_bus) | ZERO,`valid` HARDCODED false at init |
| `MAG_Bus` | `ins_stub` | RB-07 cadence per §4.6 | mandatory(多旋翼启用 mag) | [A3 §4.3.7.2](../A-architecture/A3-module-boundaries.md#4372-mag_bus典型-cadence--10-msvariant-conditional) + [B1 §4.8.2](../B-contracts/B1-bus-inventory.md#482-mag_bus) | ZERO,`valid` HARDCODED false at init |
| `Barometer_Bus` | `ins_stub` | RB-07 cadence per §4.6 | mandatory | [A3 §4.3.7.3](../A-architecture/A3-module-boundaries.md#4373-barometer_bus典型-cadence--10-ms) + [B1 §4.8.3](../B-contracts/B1-bus-inventory.md#483-barometer_bus) | ZERO,`valid` HARDCODED false at init |
| `GPS_uBlox_Bus` | `ins_stub` | RB-07 cadence per §4.6 | mandatory(GPS-denied 由 H4 故障注入) | [A3 §4.3.7.4](../A-architecture/A3-module-boundaries.md#4374-gps_ublox_bus典型-cadence--100200-ms) + [B1 §4.8.4](../B-contracts/B1-bus-inventory.md#484-gps_ublox_bus) | ZERO,`valid` HARDCODED false,`fix_type` HARDCODED no-fix(per A6 §4.4.2 + B2 `UbloxFixType.NO_FIX`) |
| `AirSpeed_Bus` | (variant) `ins_stub` 不消费;Phase 2 多旋翼 valid 始终 false | RB-07 cadence per §4.6 | optional / variant-conditional(FW/VTOL only;多旋翼默认禁用,但字段保留) | [A3 §4.3.7.5](../A-architecture/A3-module-boundaries.md#4375-airspeed_bustypical-cadence--1020-msvariant-conditional固定翼--vtol-启用多旋翼默认禁用) + [B1 §4.8.5](../B-contracts/B1-bus-inventory.md#485-airspeed_bus) | ZERO,`valid` HARDCODED false(多旋翼 Phase 2 永远 false,per A3 §4.3.7.5) |

注:Plant 内部 1 ms 主循环按 cadence 触发各传感器输出更新(RB-07),不引入内部多速率(架构 v1 §12.4 / A4 §4.4 RB-07 决策)。具体 cadence 数值在 §4.6 锁定。

### 4.5 各组件功能行为(per-component functional behavior)

本节给出每个 §4.1 组件的 (a) 输入 / 输出信号、(b) 关键关系(伪数学,标 `pseudo`)、(c) 保真度条件分支。结构落地(块图、子系统层次、共享归属)由 [C2](C2-plant-structural.md);多旋翼几何具体常数与电机模型常数由 [C3](C3-multicopter-leaf.md)。

#### 4.5.1 C1.P1 刚体 6-DoF(L4)

**输入信号**:`F_b_total_n`(体系合力,来自 P3 推力 + P2 气动 + P5 扰动)/ `M_b_total_nm`(体系合力矩,同源)/ `mass_kg`(`PLANT_PARAM.01` echo)/ `inertia_b_kgm2`(`PLANT_PARAM.02`)/ `gravity_ned_mps2`(`Environment_Info_Bus`)。

**输出信号**:`pos_ned_m` / `vel_ned_mps` / `acc_ned_mps2` / `acc_b_mps2`(specific force,含重力)/ `quat_ned_to_b` / `euler_ned_to_b_rad`(由 quat 派生)/ `ang_rate_b_radps` / `ang_acc_b_radps2`(可选输出,见 [B1 §4.5.1](../B-contracts/B1-bus-inventory.md#451-plant_states_bus) order 8 optional)/ `on_ground`(布尔,初值 HARDCODED true,由触地检测翻转;触地检测算法粒度由 C2 决定)。

**关键方程**(L4,无小角度近似):

```pseudo
# 平动(NED 系)
acc_ned_mps2 = R_b_to_ned(quat_ned_to_b) * (F_b_total_n / mass_kg) + gravity_ned_mps2
vel_ned_mps  = integrate(acc_ned_mps2)              # 积分器由 C4 决定
pos_ned_m    = integrate(vel_ned_mps)

# 体系比力(IMU 用):减去重力的反作用,得 specific force
acc_b_mps2   = R_ned_to_b(quat_ned_to_b) * (acc_ned_mps2 - gravity_ned_mps2)

# 转动(体系)
ang_acc_b_radps2 = inv(inertia_b_kgm2) * (M_b_total_nm - cross(ang_rate_b_radps, inertia_b_kgm2 * ang_rate_b_radps))
ang_rate_b_radps = integrate(ang_acc_b_radps2)
quat_dot         = 0.5 * Omega(ang_rate_b_radps) * quat_ned_to_b
quat_ned_to_b    = integrate(quat_dot);  normalize(quat_ned_to_b)   # 单位化策略 by C4
```

**保真度条件分支**:Phase 2 锁定 L4。无 L1–L3 降级路径(若降级到 L3 = 小角度近似 / 欧拉积分,会破坏闭环算法验证有效性,由 C4 数值设计禁止)。

**触地行为**(L4 必备):`on_ground` 由 `pos_ned_m.z >= ground_z + ε` 翻转;`vel_ned_mps.z` 在触地时被钳位到非负(避免穿地);姿态在地面被钳位到水平(可选,由 C2 决定)。

#### 4.5.2 C1.P2 气动(L2 默认)

**输入信号**:`vel_ned_mps`(P1 输出)/ `quat_ned_to_b` / `wind_vel_ned_mps`(`Environment_Info_Bus`)/ `air_density_kgpm3`(配置)/ `drag_lin_coef_b`(`PLANT_PARAM.12`)/ `drag_quad_coef_b`(`PLANT_PARAM.13`)。

**输出信号**:`F_aero_b_n`(体系气动力,贡献到 P1 的 `F_b_total_n`)。多旋翼 Phase 2 默认 `M_aero_b_nm = 0`(气动力矩在 L2 下忽略,推迟到 L3)。

**关键方程**(L2):

```pseudo
# 体系下相对风速
vel_rel_ned_mps = vel_ned_mps - wind_vel_ned_mps
vel_rel_b_mps   = R_ned_to_b(quat_ned_to_b) * vel_rel_ned_mps

# 阻力(线性 + 二次,逐轴解耦,体系下)
F_aero_b_n[i] = -(drag_lin_coef_b[i] * vel_rel_b_mps[i]
                 + drag_quad_coef_b[i] * vel_rel_b_mps[i] * abs(vel_rel_b_mps[i]))   for i in {x, y, z}
```

**保真度条件分支**:
- L1(降级,通常用于 unit-test):`F_aero_b_n = 0`(无气动)。
- L2(默认):上式。
- L3(升级):加地效项 `F_ground_effect = f(pos_ned_m.z, prop_wash)`;气动力矩纳入。由 C2 / C3 评估。
- L4:推迟到 fixed-wing。

#### 4.5.3 C1.P3 推进 / 电机(L2 默认)

**输入信号**:`motor_cmd[]`(`Control_Out_Bus`,4 通道多旋翼 Phase 2)/ `motor_time_const_s`(`PLANT_PARAM.10`)/ `motor_thrust_max_n`(`PLANT_PARAM.08`)/ `motor_thrust_curve_coef`(`PLANT_PARAM.11`,`[a0, a1, a2]`)/ `motor_torque_const_nmpn`(`PLANT_PARAM.09`)/ `motor_pos_b_m`(`PLANT_PARAM.06`)/ `motor_dir`(`PLANT_PARAM.07`)/ `arm_length_m`(`PLANT_PARAM.04`)。

**输出信号**:`F_motor_b_n`(体系合力,贡献到 P1 `F_b_total_n`)/ `M_motor_b_nm`(体系合力矩,贡献到 P1 `M_b_total_nm`)/ `motor_rpm[]`(`Plant_States_Bus.motor_rpm`,4 元素 Phase 2)。

**关键方程**(L2):

```pseudo
# 一阶电机响应:由命令到实际归一化推力
for k in {0..motor_count-1}:
    u_actual[k] = first_order_filter(motor_cmd[k], tau=motor_time_const_s)

# 推力曲线(单电机)
thrust_per_motor_n[k] = motor_thrust_max_n * (a0 + a1 * u_actual[k] + a2 * u_actual[k]^2)
torque_per_motor_nm[k] = motor_dir[k] * motor_torque_const_nmpn * thrust_per_motor_n[k]

# 几何合成(多旋翼:推力沿体系 -z;力矩 = 位置 × 推力 + 反扭矩)
F_motor_b_n  = sum_over_k( [0, 0, -thrust_per_motor_n[k]] )
M_motor_b_nm = sum_over_k( cross(motor_pos_b_m[k], [0, 0, -thrust_per_motor_n[k]])
                         + [0, 0, torque_per_motor_nm[k]] )

# 转速(用于 truth state 公布;具体由 thrust 反算的近似)
motor_rpm[k] = sqrt(thrust_per_motor_n[k] / k_t) * 60 / (2*pi)   # k_t 由 leaf C3 给出
```

**保真度条件分支**:
- L1(降级):零延迟 `u_actual = motor_cmd`;线性推力 `thrust = motor_thrust_max_n * motor_cmd`。
- L2(默认):上式。
- L3(升级):加电池电压 `V_batt` 状态,`thrust_max_n = f(V_batt)`(电池 sag);推迟到 Phase 4。
- L4:含 ESC 控制延迟与电流环;推迟到 SIH。

#### 4.5.4 C1.P4 传感器合成(L3 默认,详见 §4.6)

详细见 §4.6 测量模型。本节仅给出顶层数据流:

```pseudo
# IMU
gyr_b_radps  = ang_rate_b_radps + bias_gyro_radps + noise_gauss(sigma_gyro)
acc_b_mps2   = (P1 specific force) + bias_acc_mps2 + noise_gauss(sigma_acc)

# MAG / Baro / GPS / AirSpeed:类似(truth + bias + noise),per §4.6
```

**保真度条件分支**:由 PLANT_PARAM 噪声 σ 与偏置类配置(σ=0 + bias=0 即 L1;σ>0 固定偏置即 L2;σ>0 + 随机游走偏置即 L3,Phase 2 默认)。

#### 4.5.5 C1.P5 扰动注入(L1 默认 / L2 启用)

**输入信号**:`wind_vel_ned_mps`(`Environment_Info_Bus`,Phase 2 默认 = `[0,0,0]`,`PLANT_PARAM.23`)/ H4 故障注入接口(forward-cite,具体由 [H4](../H-verification/H4-fault-catalog.md))。

**输出信号**:贡献到 P2 气动的 `wind_vel_ned_mps`(已在 §4.5.2 公式)/ 直接附加到 P1 的 `F_disturbance_b_n` / `M_disturbance_b_nm`(也回写到 `Extended_States_Bus.disturbance_force_b_n` / `disturbance_torque_b_nm` 作为 trace,per A3 §4.3.6)。

**关键方程**(L2,启用湍流时):

```pseudo
# Phase 2 默认(L1):wind_total = wind_vel_ned_mps (常量)
# 启用 Dryden 湍流(L2):
wind_turbulence_ned_mps = dryden_filter(white_noise, V_air, L_scale, sigma_wind)
wind_total_ned_mps = wind_vel_ned_mps + wind_turbulence_ned_mps
# wind_total_ned_mps 注入到 P2(§4.5.2)

# 故障注入(由 H4 接口启用;C1 仅定义注入点):
F_disturbance_b_n  = h4_force_injection()    # 默认零;H4 配置非零
M_disturbance_b_nm = h4_torque_injection()   # 同上
```

**保真度条件分支**:见 §4.2.2 P5 行。

#### 4.5.6 C1.P6 环境模型(L1)

**输入信号**:`Environment_Info_Bus` 全字段(per A3 §4.3.3)。

**输出信号**:把 `gravity_ned_mps2` 喂给 P1;把 `air_density_kgpm3` 喂给 P2;把 `mag_field_ned_gauss` 喂给 P4 magnetometer;把 `air_pressure_pa` / `air_temperature_k` 喂给 P4 barometer;把 `home_lat_deg` / `home_lon_deg` / `home_alt_m` 喂给 P4 GPS(用于把 NED → LLA 转换)。

**关键方程**(L1):**全部为常量传递 + LLA 转换**。无内部建模。

```pseudo
# LLA 转换(GPS 输出用;假设 flat-earth 近似在小区域内)
gps_lat_deg = home_lat_deg + (pos_ned_m.x / R_earth_m) * (180/pi)
gps_lon_deg = home_lon_deg + (pos_ned_m.y / (R_earth_m * cos(home_lat_rad))) * (180/pi)
gps_alt_m   = home_alt_m - pos_ned_m.z
```

**保真度条件分支**:
- L1(默认):flat-earth + 常量大气 + 常量磁场。
- L2(升级):高度依赖大气模型(ISA 7 层)+ WMM 磁场;推迟到 Phase 4+。

#### 4.5.7 C1.P7 Init / reset(契约级,详见 §4.7)

不属于物理保真度。详见 §4.7。

### 4.6 传感器测量模型(sensor measurement model,functional)

本节给出每个模拟传感器的功能层模型。**所有具体 σ / 偏置数值 / cadence 数值** 由 [B3 §4.3 PLANT_PARAM](../B-contracts/B3-parameter-schema.md#43-plant_param-schema) 拥有(C1 仅 echo 字段名);本节锁定**模型类**(noise class / bias class)与**架构 v1 §12.4 委托给 C1 的 cadence 数值**(per [A4 §4.4 RB-07](../A-architecture/A4-rate-boundaries.md#rb-07-plant-1-ms--慢传感器输出plant-模块内部) 委托)。

#### 4.6.1 各传感器测量模型表(L3 default)

| Bus | Truth source field(`Plant_States_Bus`) | 噪声模型类 | 偏置模型类 | Latency | Cadence(C1 锁定,per RB-07) | Validity flag 行为 |
|---|---|---|---|---|---|---|
| `IMU_Bus.gyr_b_radps` | `ang_rate_b_radps` | Gaussian white(σ from PLANT_PARAM.14) | Random walk(初值 PLANT_PARAM.16) | 0(同 step) | **1 ms**(与 Plant 同步,无下采样)| init = false(HARDCODED);running = `true` after `T_imu_warmup_s` (TBD by C2;Phase 2 占位 0);H4 fault 可强制 false |
| `IMU_Bus.acc_b_mps2` | `acc_b_mps2`(P1 specific force) | Gaussian white(σ from PLANT_PARAM.15) | Random walk(初值 PLANT_PARAM.17) | 0 | 1 ms | 同上 |
| `MAG_Bus.mag_b_gauss` | derived from `quat_ned_to_b` × `Environment_Info_Bus.mag_field_ned_gauss` | Gaussian white(σ from PLANT_PARAM.18) | Constant(Phase 2)/ Random walk(L4 升级) | 0 | **10 ms** | init false → after warmup true;H4 可禁 |
| `Barometer_Bus.pressure_pa` | derived from `pos_ned_m.z` × ISA 标准压高关系 + `Environment_Info_Bus.air_pressure_pa`(home 海拔基准) | Gaussian white(σ from PLANT_PARAM.19) | Constant(Phase 2) | 0 | **20 ms**(50 Hz;典型 MS5611 / BMP280)| init false → after warmup true |
| `Barometer_Bus.altitude_m`(derived) | from `pressure_pa` 反算 | 派生自 `pressure_pa` 噪声 | 派生 | 0 | 20 ms | 同上 |
| `GPS_uBlox_Bus.lat_deg` / `lon_deg` / `alt_m` | derived from `pos_ned_m` × `Environment_Info_Bus.home_*`(per §4.5.6 LLA 转换) | Gaussian white(σ from PLANT_PARAM.20) | Constant zero(Phase 2;真实 GPS 无静态偏置)| **加速延迟接受**:200 ms(典型 uBlox 5 Hz)| **200 ms**(5 Hz)| GPS lock 模型:init `valid=false` & `fix_type=NO_FIX`;经过 `T_gps_lock_s`(TBD by C2;Phase 2 占位 5 s)后翻转 `valid=true` & `fix_type=GPS_3D_FIX`(per B2 `UbloxFixType`);H4 GPS-denied 可强制回 false |
| `GPS_uBlox_Bus.vel_ned_mps` | derived from `vel_ned_mps` | Gaussian white(σ from PLANT_PARAM.21) | Constant zero | 同 lat/lon/alt | 200 ms | 同上 |
| `GPS_uBlox_Bus.num_sat` / `hdop` / `vdop` | constant 占位(Phase 2 不模拟卫星几何)| N/A | N/A | N/A | 200 ms | init `num_sat=0` / `hdop=99`;after lock `num_sat=12` / `hdop=1.0`(具体值 by C2) |
| `AirSpeed_Bus.airspeed_mps` | (multicopter Phase 2 不消费;`valid` 永远 false) | N/A(多旋翼)| N/A | N/A | N/A(字段 ZERO) | 永远 false(per A3 §4.3.7.5) |

**Cadence 决策依据(闭合 [A4 §4.4 RB-07](../A-architecture/A4-rate-boundaries.md#rb-07-plant-1-ms--慢传感器输出plant-模块内部) 委托)**:
- IMU = **1 ms**:与 firmware ICM-20689 / MPU9250-class 1 kHz 主时钟匹配,且与 Plant 主循环同 rate(无下采样)
- MAG = **10 ms**:典型 HMC5883L / IST8310 / RM3100 100 Hz I2C cadence
- Baro = **20 ms**:典型 MS5611 / BMP280 oversampling 50 Hz;与 ins_stub 10 ms 公布(per A4 §4.5)的 RB-07 边界对齐(20 ms 是 1 ms 与 10 ms 的整数倍,满足 A4 R3 single-rate-aligned)
- GPS = **200 ms**:典型 uBlox NEO-M8N 5 Hz;远小于 Plant 1 ms,Plant 内部计数器实现
- AirSpeed = N/A(多旋翼禁用;字段保留为 ZERO)

所有 cadence 均为 1 ms 的整数倍(per [A4 §4.4 RB-07 R3 single-rate-aligned](../A-architecture/A4-rate-boundaries.md#rb-07-plant-1-ms--慢传感器输出plant-模块内部))。

#### 4.6.2 噪声 / 偏置模型类定义

| 类名 | 定义(伪) | 用途 |
|---|---|---|
| **Zero**(L1) | `noise = 0; bias = 0` | unit test / 理想化 baseline |
| **Gaussian-white + Constant-bias**(L2) | `noise[k] = N(0, σ); bias = bias_init`(标量 / 向量,固定) | 一阶传感器近似 |
| **Gaussian-white + Random-walk-bias**(L3,Phase 2 默认 IMU) | `noise[k] = N(0, σ); bias[k+1] = bias[k] + N(0, σ_bw * sqrt(dt))` | EKF / ECL 设计依据(架构 v1 §12.3 firmware INS 的 Gauss-Markov 模型) |
| **Colored-noise + drift**(L4) | `noise = filter(N(0, σ), correlation_time); bias = drift(t, T)` | SIH-class |

#### 4.6.3 RNG seeding(forward-cite to H3)

所有传感器噪声 / 偏置随机游走的随机数源由 [H3 RNG seeding 契约](../H-verification/H3-regression-baseline.md) 管理。本文件**不**自定 RNG 接口,仅承诺:Plant 不在 init 中读取 wall-clock,所有随机数源种子由 PARAM 路径传入(满足 A6 §4.6.1 "禁止反模式:在 init 中由 RNG 决定的初值")。

#### 4.6.4 Validity flag 通用行为契约

1. **init 后**:全部传感器 `valid = false`,`fix_type` / `num_sat` 等离散字段 = 安全态(per A6 §4.4.2 ZERO + B2 enum 安全态)。
2. **Warmup 完成后**:`valid` 翻转 true。Warmup 时长由 PARAM(C2 决定具体名);Phase 2 默认 IMU/MAG/Baro = 0(无 warmup)、GPS = 5 s(模拟卫星捕获)。
3. **Reset 后**:回到 init 状态;warmup 重新计时。
4. **故障注入(H4 forward-cite)**:H4 可强制 `valid = false`(传感器失效)或维持 `valid = true` 但注入异常数值(传感器漂移 / spike);C1 仅约定注入点,具体故障类由 [H4](../H-verification/H4-fault-catalog.md)。

### 4.7 Init / reset 行为(per [A6](../A-architecture/A6-init-reset-contract.md))

C1 把 `Plant_States_Bus` / `Extended_States_Bus` / 各传感器 bus 的 init 后稳态值映射到 [A6 §4.4.1 类别](../A-architecture/A6-init-reset-contract.md#441-类别定义)。

#### 4.7.1 Plant_States_Bus 字段类别映射

| 字段 | A6 类别 | 数值来源 | 备注 |
|---|---|---|---|
| `pos_ned_m` | **PARAM** | `States_Init_Bus.pos_ned_m` echo(per A3 §4.3.5;ultimate source = `PLANT_PARAM.29 pos_ned_m_init`)| Phase 2 默认 `[0,0,0]` |
| `vel_ned_mps` | **PARAM** | `States_Init_Bus.vel_ned_mps`(`PLANT_PARAM.30`) | 默认 `[0,0,0]` |
| `acc_ned_mps2` | **ZERO** | — | init 时无加速度;第一 step 后由 P1 计算 |
| `acc_b_mps2` | **ZERO** | — | 同上 |
| `quat_ned_to_b` | **PARAM** | `States_Init_Bus.quat_ned_to_b`(`PLANT_PARAM.31`)| 默认 `[1,0,0,0]` identity |
| `euler_ned_to_b_rad` | **PARAM(derived)** | derived from `quat_ned_to_b` | |
| `ang_rate_b_radps` | **PARAM** | `States_Init_Bus.ang_rate_b_radps`(`PLANT_PARAM.32`)| 默认 `[0,0,0]` |
| `ang_acc_b_radps2` | **ZERO** | — | optional 字段 |
| `mass_kg` | **PARAM** | `PLANT_PARAM.01` echo | constant 整段仿真(Phase 2 不模拟燃料消耗) |
| `inertia_b_kgm2` | **PARAM** | `PLANT_PARAM.02` echo | constant |
| `motor_rpm[]` | **ZERO** | — | per A6 §4.4.2 "电机/舵面状态 ZERO 多旋翼默认";A6 §4.4.2 显式允许 PARAM 例外但 Phase 2 选 ZERO |
| `on_ground` | **HARDCODED true** | constant true at init | per A6 §4.4.2 默认在地面 |
| `timestamp` | **ZERO** | — | per A7 |

#### 4.7.2 Extended_States_Bus 字段类别映射

per A6 §4.4.2:全字段 **ZERO** 默认。Phase 2 多旋翼无例外。`disturbance_force_b_n` / `disturbance_torque_b_nm` trace 字段在 init 后 = ZERO,扰动启用时由 P5 实时写入。

#### 4.7.3 各传感器 bus 字段类别映射

per A6 §4.4.2:全字段 **ZERO** 默认;`valid` **HARDCODED false**;`fix_type`(GPS) **HARDCODED no-fix-equiv**(B2 `UbloxFixType.NO_FIX`,具体数值由 B2)。详细已在 §4.6.4 / §4.4 表给出。

#### 4.7.4 Reset 协议遵循(per A6 §4.5.2)

- **Plant 不响应 `FMS_Out_Bus.reset`**(per A6 §4.5.2 关键约束 1:"global reset 不联动 Plant reset")。
- **Plant reset 仅由 harness 显式触发**(MIL 路径下 = TS-MIL-HARNESS;firmware 路径下不存在,因为 Plant 不在 firmware 飞行运行期)。
- Reset 后,所有 §4.7.1–§4.7.3 类别契约必须在**第一次 step 之前**到位(per A6 §4.4 定义)。
- Reset 不能引入 RNG 决定的初值(per A6 §4.6.1 反模式禁令);所有传感器 noise / bias 的随机游走状态在 reset 时回到 PARAM 初值(`PLANT_PARAM.16` / `.17` 等)。

#### 4.7.5 Plant_init 契约(per A6 §4.2.3 Plant 行)

- `Plant_init(States_Init_Bus)` 必须使所有 §4.7.1 PARAM 字段获得 `States_Init_Bus` 提供的初值,所有 ZERO / HARDCODED 字段获得契约值。
- `Plant_init` 不得调用 `FMS_step` / `Controller_step`(per A6 §4.6.1 反模式禁令)。
- `Plant_init` 不读取外部文件 / 环境变量 / wall-clock(per A6 §4.6.1)。

### 4.8 扰动注入点(disturbance injection points,functional 定位)

本节仅给出**功能层注入点**(注入到哪个物理量);结构层(块图、Mux 位置、shared block 归属)由 [C2](C2-plant-structural.md) + [A8 §4.3.1 `Disturbance_Injector_Shell`](../A-architecture/A8-shared-library-roster.md#431-modelplantshared--plant-共享架构块)。

| 注入点 ID | 注入到的物理量 | 来源 | trace 字段(可观测性)|
|---|---|---|---|
| **DI-01** | P2 气动力的 `wind_vel_ned_mps`(替代 `Environment_Info_Bus` 静态风) | `Environment_Info_Bus.wind_vel_ned_mps` 配置量 + Dryden 湍流(L2 启用时)+ H4 风扰动场景 | 写入到 P5 的内部 `wind_total_ned_mps`,可由 G3 日志记录 |
| **DI-02** | P1 体系合力 `F_b_total_n` 的"额外扰动力"项 | H4 接口(forward-cite) | `Extended_States_Bus.disturbance_force_b_n`(per A3 §4.3.6) |
| **DI-03** | P1 体系合力矩 `M_b_total_nm` 的"额外扰动力矩"项 | H4 接口 | `Extended_States_Bus.disturbance_torque_b_nm` |
| **DI-04** | P4 各传感器测量值的 `noise` / `bias` 项 | `PLANT_PARAM.14–.22`(已设计噪声 σ)+ H4 noise scaling / bias drift 故障 | 各传感器 bus(噪声本身就是观测) |
| **DI-05** | P3 电机推力的"电机失效"乘子(单电机 `thrust_per_motor_n[k] *= scale_k`) | H4 motor failure 故障类(forward-cite)| `Plant_States_Bus.motor_rpm[k]` 间接反映;G3 日志 |
| **DI-06** | P6 环境模型的 GPS lock 状态(强制 unlock) | H4 GPS-denied 故障类 | `GPS_uBlox_Bus.valid` / `fix_type` |

**注**:DI-01 是 PARAM-driven(常态)+ H4-driven(场景);DI-02 / DI-03 / DI-04 / DI-05 / DI-06 是 H4-driven(故障);DI-04 中"基线噪声"由 PARAM(常态 L3)+ "异常噪声"由 H4。具体接口签名(数据通路 / mux 位置)由 C2 + H4 协同定义。

### 4.9 Open issues / 委托清单

下列项 C1 故意不裁决(超出本工作项范围或推迟到下游):

| Open ID | 议题 | 委托给 | 推迟原因 |
|---|---|---|---|
| C1-OQ1 | Plant 内部子系统层次 / 共享 vs leaf 边界 / 库块清单 | [C2](C2-plant-structural.md) | 结构设计是 C2 范围 |
| C1-OQ2 | 多旋翼几何(arm length, rotor positions, mass, inertia)/ 电机模型常数 / 数值默认 | [C3](C3-multicopter-leaf.md) | leaf 范围,且 [B3 §4.3.2](../B-contracts/B3-parameter-schema.md#432-字段表) 已给出 PARAM 字段名;C3 复核数值 |
| C1-OQ3 | 数值积分器选择(RK4 / Heun / FE)/ 步长 / quat 单位化策略 / reset 时积分器内部状态 | [C4](C4-numerics.md) | 数值范围;C4 在 PLANT_PARAM.33 / .34 落地 |
| C1-OQ4 | 固定翼 / VTOL 气动子模型 | [A5 Phase 5 路径](../A-architecture/A5-variant-strategy.md) | 00-design-plan §5 排除 |
| C1-OQ5 | INS 估计算法 | firmware-owned(架构 v1 §3 / §12.3) | 不在模型仓 |
| C1-OQ6 | INS_stub 如何从 Plant_States_Bus 派生 INS_Out_Bus(噪声 / 偏置 / 延迟具体接口)| [F1](../F-ins-contract/F1-ins-stub-functional.md) / [F2](../F-ins-contract/F2-ins-stub-structural.md) | C1 仅承诺发布 truth + 模拟传感器 |
| C1-OQ7 | 故障注入分类 / 触发参数 / 接口签名 | [H4](../H-verification/H4-fault-catalog.md) | 故障目录范围 |
| C1-OQ8 | RNG seeding 接口 / 种子矩阵 | [H3](../H-verification/H3-regression-baseline.md) | 回归基线 / 种子契约范围 |
| C1-OQ9 | 触地检测细节(hysteresis 阈值 / 速度钳位实现) | [C2](C2-plant-structural.md) + [C3](C3-multicopter-leaf.md) | 结构 + leaf 范围 |
| C1-OQ10 | IMU/MAG/Baro warmup 时长(`T_imu_warmup_s` / `T_gps_lock_s` 等)是否要 PARAM 暴露 | [C2](C2-plant-structural.md) + [B3 增补](../B-contracts/B3-parameter-schema.md) | 若 C2 决定 PARAM 化,需向 B3 提交 PARAM 增补请求 |

## 5. 已知风险与悬而未决问题

- **R-1 保真度等级与 ins_stub 现实性互锁**(架构 v1 §17 risk 4)
  - 影响:F1 / F2 ins_stub 设计、H1 验证场景
  - 风险:若 §4.6 传感器噪声模型 (L3) 已经"足够现实",ins_stub 在 MIL 中可能复用同一噪声源;若 ins_stub 反而引入第二层噪声,会双重失真。需 F1 / F2 设计时显式声明 ins_stub 是否"trust Plant 传感器输出 as-is" 还是"在 truth 基础上再次注入"。
  - 处置:本文件 §4.6 已锁定 P4 = L3(Gaussian + random-walk bias)作为 Plant 内噪声基线;F1 在自身退出条件中决定是否再次注入。
- **R-2 慢传感器 cadence 数值与 firmware 实际 cadence 不一致风险**
  - 影响:SIH 阶段(架构 v1 §15.3)发现 MIL cadence 与真实传感器 cadence 不符
  - 处置:§4.6.1 cadence 决策已记录每个传感器对应的典型硬件型号;SIH 启动前由 [H1 §SIH 重跑](../H-verification/H1-scenario-catalog.md)校准;若校准失败,本文件 cadence 数值需修订(变更日志记录)
- **R-3 气动 L2 默认在低高度 / 近地稳定性不足**
  - 影响:H1 takeoff/landing 场景可能出现非物理震荡
  - 处置:首次 H1 验证若发现近地不稳,触发 §4.5.2 升级到 L3(地效模型);C2 / C3 在结构上预留升级钩子
- **R-4 触地模型 L4 必备但未给详细算法**
  - 影响:C2 / C3 实现时可能各自发明算法,导致 leaf 之间不一致
  - 处置:C1-OQ9 已标委托给 C2 + C3;C2 应锁定共享触地块的算法接口
- **R-5 RNG seeding 接口尚未契约化**
  - 影响:若 H3 RNG 契约迟到,Plant 无法实现 deterministic 仿真
  - 处置:Wave 11(H3 启动)前 Plant 实现可暂用 PARAM 种子占位;H3 reviewed 后回填;§4.6.3 已 forward-cite
- **R-6 GPS LLA 转换 flat-earth 近似在大区域(> 10 km)失真**
  - 影响:大范围 mission 场景失真
  - 处置:Phase 2 多旋翼 takeoff/hover/landing 场景半径 ≤ 100 m,误差忽略;若 H1 引入大范围航点,触发 §4.5.6 升级 L2(WGS84 椭球转换)
- **R-7 PLANT_PARAM 数值占位 vs firmware tip 一致性**(per [B3 §4.3 footnote](../B-contracts/B3-parameter-schema.md#43-plant_param-schema))
  - 影响:契约 byte-level 验证(B4 / I3)
  - 处置:B3 已标 `(B4 verify against firmware tip)`;C1 不重复声明,仅引用

## 6. 退出条件复核

对照 [`00-design-plan.md` §4.C C1 行](../00-design-plan.md):**"仿真物理范围(刚体/气动/电机/传感器/扰动)与各项物理保真度等级"**。

| # | 退出条件原文(拆分为 2 个 atomic 子条件)| 本文档依据 | 状态 |
|---|---|---|---|
| (a) | 仿真物理范围(刚体/气动/电机/传感器/扰动)枚举完成 | §4.1 表给出 7 类组件:**C1.P1 刚体 6-DoF / C1.P2 气动 / C1.P3 推进电机 / C1.P4 传感器 / C1.P5 扰动 / C1.P6 环境 / C1.P7 init/reset**,完全覆盖退出条件提到的 5 类(刚体 / 气动 / 电机 / 传感器 / 扰动)+ 任务额外要求的环境与 init/reset(架构 v1 §12.4 6 阶段 + A6 init/reset)| **满足** |
| (b) | 各项物理保真度等级给出 | §4.2.1 定义 4 级 ladder;§4.2.2 表给出**每个 §4.1 组件的 Phase 2 默认 level + 分级理由 + 升级路径**:P1=L4 / P2=L2 / P3=L2 / P4=L3 / P5=L1→L2 / P6=L1 / P7=L4(契约级)| **满足** |

辅助 (但非退出条件强制):

| # | 辅助检查 | 本文档依据 | 状态 |
|---|---|---|---|
| (c) | A1 §5.1.7 "optional 传感器准则未定" 闭合 | §4.4 表 + §4.6.1:Phase 2 多旋翼 mandatory = IMU / MAG / Baro / GPS;optional / variant-conditional = AirSpeed(永远 false 多旋翼);准则 = "ins_stub 是否消费 + 多旋翼是否物理装备" | **满足** |
| (d) | A1 §12.4 "各阶段保真度等级未定" 闭合(C 系列覆盖,C1 部分)| §4.2.2 完整表 | **满足** |
| (e) | A4 §4.4 RB-07 "慢传感器 cadence 由 C1 决定" 委托闭合 | §4.6.1 表给出 IMU=1ms / MAG=10ms / Baro=20ms / GPS=200ms / AirSpeed=N/A;§4.6.1 footnote 给出依据 | **满足** |
| (f) | C1 ⇢ F1 信息流(per [关系图 §4.6](../01-design-relationships.md#46-f-区ins-契约消费))| §4.4 输出契约 + §4.6 测量模型 + §4.7 init 行为 + §4.9 C1-OQ6;F1 启动只需读 §4.4 + §4.6 即可 | **满足** |

## 7. 下游影响

按 [`01-design-relationships.md` §4.3 / §4.6](../01-design-relationships.md):

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [C2 Plant 结构设计](C2-plant-structural.md) | → (强前置) | §4.1 物理范围(决定子系统数量与边界)+ §4.5 各组件功能行为(决定块图骨架)+ §4.7 init/reset(决定 reset 入口结构);C1-OQ1 / OQ9 / OQ10 委托项 |
| [C3 Plant 多旋翼 leaf](C3-multicopter-leaf.md) | → (经 C2 间接) | §4.5.3 推进电机方程结构(C3 填具体参数与混控);§4.5.1 触地行为(C3 填地形参数);C1-OQ2 / OQ9 委托项 |
| [C4 Plant 数值与积分](C4-numerics.md) | → (强前置) | §4.5.1 刚体方程(决定积分器需求 — quat 单位化、多状态耦合)+ §4.2 P1=L4(决定数值精度需求,禁止小角度近似);C1-OQ3 委托项 |
| [F1 ins_stub 功能设计](../F-ins-contract/F1-ins-stub-functional.md) | ⇢ (信息流) | §4.4 Plant 输出契约(F1 输入)+ §4.6 测量模型(F1 决定是否复用 / 双重注入)+ §4.7.4 reset 协议(F1 在 ins_stub reset 时遵循);C1-OQ6 委托项 |
| [G1 MIL 顶层结构](../G-harness/G1-mil-toplevel.md) | ⇢ (经 C2/C4 间接) | §4.3 / §4.4 决定 G1 在 harness 中接入 Plant 的 bus 集 |
| [H1 验证场景目录](../H-verification/H1-scenario-catalog.md) | ⇢ (间接) | §4.2.3 推迟项决定 H1 不能假设 Phase 2 已具备地效 / 电池 sag / 全谱湍流 |
| [H4 故障注入目录](../H-verification/H4-fault-catalog.md) | ⇢ (间接) | §4.8 注入点 ID(DI-01..DI-06)= H4 在功能层的目录起点;C1-OQ7 委托 |

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-08 | C1 author | 初稿;闭合退出条件 (a)(b);锁定 §4.6 慢传感器 cadence(委托自 A4 RB-07);7 类物理范围、4 级 ladder、Phase 2 默认 L1/L2/L3/L4 矩阵 |

## Self-check

- [x] frontmatter 完整,字段值合法
- [x] 上游文档全部存在且 status ≥ reviewed:架构 v1(reviewed)、A1–A8(reviewed)、B1–B3(reviewed);无同 batch 兄弟
- [x] 退出条件逐条复核完成,每条均给出依据(§6 表 (a)(b) 满足 + 辅助 (c)–(f) 满足)
- [x] 引用路径全部可点击访问(均为相对路径,指向已存在文档)
- [x] 不存在 RULES §5 禁则中的内容(无 .slx 截图、无可执行 .m、无重复 firmware 实现细节、无 PR/branch 名、无重复 bus/enum 字段表 — 均引用 A3/B1)
- [N/A] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记 — **本文 contract_impact=no,N/A**
- [N/A] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径 — **本文不镜像 firmware 契约(只引用 A3 / B1 / B3 已镜像项,镜像义务在 B 区);N/A**
- [x] 下游影响已沿关系图识别完毕(§7 表覆盖 C2 / C3 / C4 / F1 / G1 / H1 / H4 全部出边)
- [x] 文档不超出本工作项范围(无越权设计):未定义结构(C2)、未定义数值(C4)、未定义 leaf 数值(C3)、未重定义 bus(B1)、未重定义 enum(B2)、未重定义 PARAM 数值(B3);不在范围项已显式列在 §2
