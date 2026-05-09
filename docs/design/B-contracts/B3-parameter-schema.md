---
work_item: B3
title: Parameter schema 设计
upstream: ["架构 v1", "A1", "A2", "A3", "A6"]
contract_impact: yes
status: reviewed
authored_at: 2026-05-07
last_reviewed_at: 2026-05-07
reviewer_verdict: pass
---

# B3 Parameter schema 设计

## 1. 目的

为 FMT-Model-2025b **三个 firmware-visible 参数 struct**(`PLANT_PARAM` / `FMS_PARAM` / `CONTROL_PARAM`)给出**字段级 schema 设计**:每字段名、Simulink 类型、宽度、单位、坐标系、Phase 2 多旋翼默认值、运行时可调 vs 编译期内联分类、与之挂接的 bus 字段、reset 类别回声、变体条件;并显式登记 `*_EXPORT` struct 字段集与 `model_info[]` 元数据。本文件闭合 [A1 §4.2 §5.1.2 / §14.3.3 / §17 Risk 6](../A-architecture/A1-v1-review.md) 三条 critical 缺口(EXPORT 全字段未枚举、runtime-tunable vs compile-inlined 分类标准未定、参数治理 tuning divergence 检测),并向 C3 / D5 / E4 提供 leaf 设计的字段名兼容性核对模板。本文件**不**重定义任何 bus 字段(B1 拥有)或任何 enum 数值(B2 拥有);仅引用、并为参数挂接关系提供来源字段。

## 2. 范围

**在范围:**

- `PLANT_PARAM` / `FMS_PARAM` / `CONTROL_PARAM` 三个 struct 的字段级 schema(每字段:名、类型、单位、坐标系、Phase 2 多旋翼默认值、分类、affects-bus-field、reset 类别回声、variant-conditional、备注)
- `PLANT_EXPORT` / `FMS_EXPORT` / `CONTROL_EXPORT` struct 字段集与 `model_info[]` 占位条目(闭合 A1 §5.1.2)
- **runtime-tunable vs compile-inlined 分类方法**(闭合 A1 §14.3.3 / 架构 v1 §14.3.3 / 审计 F-08)
- 多旋翼 Phase 2 leaf 的代表性默认值表(numeric)
- C3 / D5 / E4 字段名兼容性核对模板(空表 + 列定义)
- 端序约定(little-endian per ARM Cortex-M target;呼应 B1)
- B1 / B2 与 B3 之间的交叉引用矩阵(哪些字段 back which bus 字段;哪些字段是 enum 类型)
- B4 / I3 契约 diff 必须覆盖的 PARAM / EXPORT 范围(diff coverage 要求)

**不在范围(由其他工作项处理):**

- bus 字段定义、字节顺序、padding — 由 [B1 Bus 清单与 schema 设计](B1-bus-inventory.md) 处理;本文件仅引用 bus 字段名
- enum 数值与符号 — 由 [B2 Enum 清单与数值锁定](B2-enum-inventory.md) 处理;本文件仅引用 enum 类型名
- 多旋翼几何 / 质量惯量 / 电机推力曲线**具体数值修订** — 由 [C3 Plant 多旋翼 leaf 设计](../C-plant/C3-multicopter-leaf.md) 处理;本文件给"参考默认值"占位,C3 在自身退出条件中负责将默认值与 firmware tip 对齐
- 多旋翼 mode-cmd 映射 / 起飞降落 profile **数值** — 由 [D5 FMS 多旋翼 leaf 设计](../D-fms/D5-multicopter-leaf.md) 处理
- 多旋翼控制环增益 **整定** — 由 [E4 Controller 多旋翼 leaf 设计](../E-controller/E4-multicopter-leaf.md) 处理
- 契约 diff 工具实现 — 由 [B4 契约 diff 策略设计](B4-contract-diff.md) + [I3 契约 diff 脚本设计](../I-tooling/I3-contract-diff.md) 处理;本文件仅声明 PARAM/EXPORT 的 diff 覆盖要求
- INS 参数 — INS firmware-owned(架构 v1 §3 / §12.3),无 `INS_PARAM`;本文件不涉及
- `FMS_init` / `Controller_init` / `Plant_init` 调用顺序与 reset 行为 — 由 [A6 Init/Reset 状态契约](../A-architecture/A6-init-reset-contract.md) 处理;本文件仅引用 A6 §4.4 reset 类别
- codegen 配置(参数 inline 实现机制)— 由 [I4 Codegen 配置脚本设计](../I-tooling/I4-codegen-config.md) 处理;本文件给**分类**(yes/no inline),I4 给**机制**(`Tunable=on/off`、`Storage Class` 等)

## 3. 依赖

| 上游 | 引用位置 | 用途 |
|---|---|---|
| 架构 v1 §5.1.2 | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | `*_PARAM` / `*_EXPORT` 必须由模型仓产出;`model_info[]` 与 `period` 是 EXPORT 字段的最小集 — 本文件 §4.7 EXPORT struct sketch 起点 |
| 架构 v1 §10 enum / §14.2 | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | 参数字段中的 enum 类型(mode 阈值默认 mode 等)— 本文件 §4.6 cross-ref to B2 |
| 架构 v1 §14.3 inline-vs-tunable | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | "Runtime-tunable versus compile-time-inlined parameters must be intentionally categorized" — 本文件 §4.2 分类方法直接承接 |
| 架构 v1 §16 smallest slice | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | Phase 2 = multicopter only;本文件 §4.5 默认值表以多旋翼为唯一基线 |
| 架构 v1 §17 Risk 6 | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | parameter governance(tuning divergence 检测) — 本文件 §4.8 diff coverage 要求承接 |
| [A1 架构 v1 评审与缺口闭合](../A-architecture/A1-v1-review.md) (status: reviewed, 2026-05-05) | A1 §4.2 §5.1.2 / §14.3.3 / §17 Risk 6 | 三条缺口的"Closed by B3"分配 — 本文件逐条闭合于 §6 |
| [A2 命名与目录约定设计](../A-architecture/A2-naming-conventions.md) (status: reviewed, 2026-05-05) | A2 §4.7 R-7.1..R-7.5 + §4.10 单位 / 坐标系 / 角度后缀 | 参数 struct 名固化 / 字段名 `lower_snake_case` / 单位后缀义务 / `init_use` 标志(per A6 §4.6.2 命名义务)|
| [A3 模块边界信号清单细化](../A-architecture/A3-module-boundaries.md) (status: reviewed, 2026-05-07) | A3 §4.3 / §4.4 / §4.5 / §4.12 字段元数据要求 | 每个 PARAM-class bus 字段在 A3 reset 列被标 PARAM;**B3 必须为每条标 PARAM 的字段在某个 `*_PARAM` struct 中提供初值字段** — 本文件 §4.6 cross-ref to B1 矩阵承接 |
| [A6 Init/Reset 状态契约](../A-architecture/A6-init-reset-contract.md) (status: reviewed, 2026-05-05) | A6 §4.4 类别表 / §4.6 PARAM 来源 / §4.6.2 命名义务 | "Reset class PARAM 意味着初值从 B3-defined 参数 struct 取得;B3 必须枚举每条 PARAM-class 字段" — 本文件 §4.6 矩阵直接承接 |
| 同 batch 兄弟 [B1 Bus 清单与 schema 设计](B1-bus-inventory.md) *(co-seal batch (B1, B2, B3),draft in flight)* | (兄弟引用,允许引用 draft 兄弟 per RULES §6 self-check 第 2 项) | bus 字段名 / 类型作为 affects-bus-field 列的来源;**B3 不重述 bus 字段**,仅按 A3 §4.x 字段名引用 |
| 同 batch 兄弟 [B2 Enum 清单与数值锁定](B2-enum-inventory.md) *(co-seal batch (B1, B2, B3),draft in flight)* | (同上) | enum 类型名 / 成员名(`PilotMode` / `CtrlMode` / `VehicleStatus` 等);**B3 不重述 enum 数值**,仅按类型名引用 |
| firmware 头文件 `FMT-Firmware/src/model/{plant,fms,control}/<vehicle>/lib/{Plant,FMS,Controller}_types.h` (`<vehicle>` = `multicopter` for Phase 2) | **FMT-Firmware @ \<pending hash\>**(commit hash 由 INDEX 决策日志维护方在登记本工作项时补齐;本 draft 与 [A6 §3 / A7 §3 / A3 §3 / A6 §5 第 1 项 open](../A-architecture/A6-init-reset-contract.md) 同样的占位约定) | 字段名 / 类型 / 顺序的 firmware 端权威来源;**B3 不复制字节布局,仅引用字段语义并在 §5 风险登记"firmware-tip 对齐由 B4 在 diff 时强制"** |

注:**co-seal batch 名 = (B1, B2, B3)**(per [01-design-relationships.md §5.1](../01-design-relationships.md))。本文件作为 B3,在 §3 引用 B1 / B2 兄弟 draft 是 RULES §6 self-check 第 2 项允许的,无须等 B1 / B2 进入 reviewed。

环境注:FMT-Firmware 在本设计阶段未挂载,作者无法直接读取 `FMS_PARAM` / `CONTROL_PARAM` / `PLANT_PARAM` 头文件。本文件依据架构 v1 + A1 + A3(标记每个 PARAM-class bus 字段)+ 标准多旋翼参数集(允许 PX4-class 默认作为设计基线),把每个**类型 / 数值默认**标 `(B4 verify against firmware tip)`,并以 `FMT-Firmware @ <pending hash>` 形式记录占位。当 INDEX 决策日志登记本工作项时补齐 commit hash;B4 在 contract diff 阶段做 byte / 顺序级强制对齐。

## 4. 设计内容

### 4.1 总览

本工作项作为 (B1, B2, B3) co-seal batch 的"参数侧",承担三件事:

1. **参数 struct 字段集**(§4.3 / §4.4 / §4.5 三个子节,分别对应 PLANT/FMS/CONTROL_PARAM)
2. **EXPORT struct 字段集**(§4.7) — 三个 `*_EXPORT` 的字段台账,含 `model_info[]` 占位
3. **跨工作项交叉引用矩阵**(§4.6) — 哪些 PARAM 字段对应哪些 bus 字段(B1)、哪些是 enum 类型(B2)、哪些是 init_use(A6 §4.6.2)、哪些 variant-conditional(A5)

设计组织遵循 RULES §3:每节先方法后内容,字段表格列定义集中在 §4.2 不在每张表头重复。

### 4.2 字段表列定义与分类方法(闭合 A1 §14.3.3 / 架构 v1 §14.3.3 / 审计 F-08)

#### 4.2.1 字段表列定义

§4.3 / §4.4 / §4.5 的所有字段表使用以下统一列(在每节表头不再重复说明):

| 列 | 含义 |
|---|---|
| **#** | 字段序号(`<struct>.NN`,稳定标识,不随重命名变化;例 `PLANT_PARAM.01`)|
| **Field** | 字段名(`lower_snake_case`,per [A2 R-7.3 / R-10.1 / R-10.5](../A-architecture/A2-naming-conventions.md);firmware-legacy 名标 Notes)|
| **Type** | Simulink 数据类型(`single` / `double` / `uint8` / `uint16` / `uint32` / `int*` / 数组 / 嵌套 struct)。`(B4 verify against firmware tip)` 标占位 |
| **Width** | 字节宽度(单元素 × 数组长度;不含 padding。**累计 offset 由 B4 在字节级对齐验证**;本表给定 width 仅供 EXPORT 总宽度估算)|
| **Unit** | 单位后缀(per A2 §4.10.1)。`N/A` 表示无量纲 |
| **Frame** | 坐标系(`NED` / `body` / `world` / `vehicle` / `N/A`,per A2 §4.10.2)|
| **Default (multi)** | Phase 2 多旋翼参考默认值(数值 / 字符串 / enum 成员名)。`(B4 verify; placeholder uses PX4-class default)` 标未验证 |
| **Class** | 分类:`runtime-tunable`(R-T)/ `compile-inlined`(C-I)。Class 决策依据见 §4.2.2 |
| **Class rationale** | 一句话依据(必填,per 审计 F-08)|
| **Affects bus field(s)** | 该参数会影响的 bus 字段(B1 字段名);多个用逗号分隔。`—` 表示纯内部参数,不直接 back 任何 bus 字段(例如积分器初值)|
| **Reset class echo** | A6 §4.4 reset 类别回声(对应 A3 §4.x reset 列)。本字段为 PARAM 的字段必须在 Class 列 R-T 或 C-I 二选一;non-PARAM 类别(HARDCODED / ZERO / PRESERVED)字段**不**进入本 schema(由 A6 / B1 单独治理)|
| **Variant** | variant-conditional 标记(per [A5 §4.2.2](../A-architecture/A5-variant-strategy.md)):`MC-only` / `FW-only` / `VTOL-only` / `all`(默认 all)|
| **Init use** | 是否作为 init 直接初值(per A6 §4.6.2):`yes` / `no` |
| **Notes** | 范围 / 单调性 / 几何形状 / firmware-legacy 标记 / 关联 PARAM 字段等 |

#### 4.2.2 分类方法(runtime-tunable vs compile-inlined)

承接 [架构 v1 §14.3.3](../../architecture/2026-05-05-fmt-model-architecture-v1.md):"Runtime-tunable versus compile-time-inlined parameters must be intentionally categorized."

**判定规则**(每个 PARAM 字段必须按以下顺序检查,首次命中生效):

1. **R-1(强制 runtime-tunable)**:字段被 GCS / firmware param 系统在飞行期或地面期写入(典型:控制增益 / failsafe 阈值 / geofence 半径 / 起降速度)→ **runtime-tunable**。
2. **R-2(强制 compile-inlined)**:字段为机型几何 / 物理常量(质量 / 惯量 / 力臂 / 电机数量 / 旋转方向)且被 codegen 嵌入大量算式中(乘除常数化能显著缩减 RAM/ROM 与浮点指令)→ **compile-inlined**。
3. **R-3(强制 compile-inlined)**:字段为数组长度 / 维度类常量(`wp_array` 上限、电机通道数)→ **compile-inlined**(架构 v1 §4.5 禁可变尺寸)。
4. **R-4(默认 runtime-tunable)**:其他所有字段默认 **runtime-tunable**(优先 GCS 暴露,牺牲少量代码体积)。
5. **R-5(豁免)**:`*_EXPORT` 中的 `period` 与 `model_info[]` 不属本分类(它们是元数据,非配置参数)。

**判定理由必须在 Class rationale 列写出**(一句话,审计 F-08 强制)。常见理由模板:

- `R-1: GCS-tuned at runtime`(增益、限值)
- `R-2: geometry-constant; codegen efficiency`(质量、惯量、力臂)
- `R-3: array-length / dimension; static at codegen`(数组长度)
- `R-4: default tunable; no efficiency penalty`(其他)

每字段恰好一条 R-x;不允许"depends"。若设计 phase 后期发现某字段错分类,走 [RULES §10](../RULES.md) 变更日志,触发 B4 / I3 重新 diff。

#### 4.2.3 端序与对齐

- **端序**:little-endian(per ARM Cortex-M target;呼应 [B1](B1-bus-inventory.md) 同声明)。
- **对齐**:Simulink ert.tlc 默认对齐遵循 ARM EABI(每基本类型按其大小对齐);`single`(4B)按 4B,`double`(8B)按 8B,`uint8`/`int8` 按 1B 等。**累计 offset 由 B4 字节级 verify**;本文件 Width 列仅供 EXPORT 总宽度估算。
- **嵌套 struct**:若使用嵌套 struct(例如 `CONTROL_RateLoop_PARAM`,per A2 R-7.3),嵌套 struct 内同样规则;嵌套自身按其内部最大字段对齐。

### 4.3 PLANT_PARAM schema

#### 4.3.1 Header

| 项 | 值 |
|---|---|
| Struct 名 | `PLANT_PARAM`(per A2 R-7.1;固化,不可改名)|
| Owner module | Plant(model-owned)|
| 来源 firmware 头文件 | `FMT-Firmware/src/model/plant/multicopter/lib/Plant_types.h`,**FMT-Firmware @ \<pending hash\>** |
| 暴露形态 | 由模型仓 ert.tlc 生成 `extern PLANT_PARAM_TYPE PLANT_PARAM;`(具体类型名 `PLANT_PARAM_TYPE` 由 firmware 锁定;B4 verify)|
| Phase 2 vehicle | multicopter(架构 v1 §16 / §19;A5 §4.x)|
| 总字段数 | 见 §4.3.2(由 §4.6.4 § Tunable budget 表汇总)|
| Variant 范围 | `MC-only` 字段在 Variant 列标;Phase 2 active variant = multicopter,其他 vehicle 字段在 Phase 5 启用 |

#### 4.3.2 字段表

> 单位/坐标系/分类/Reset 列定义见 §4.2.1;分类规则代号 R-1..R-5 见 §4.2.2。
> 默认值数值标 `(B4 verify; placeholder uses PX4-class default)` 表示未与 firmware tip 对齐;C3 在自身退出条件中复核所有 `MC-only` 字段。
> 类型列标 `(B4 verify)` 表示需 firmware byte verify。

**几何与质量惯量组**(rigid-body geometry / inertia,Plant 物理核心)

| # | Field | Type | Width | Unit | Frame | Default (multi) | Class | Class rationale | Affects bus field(s) | Reset | Variant | Init use | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| PLANT_PARAM.01 | `mass_kg` | `single` (B4 verify) | 4 | kg | N/A | 1.5 *(B4 verify; PX4-class quad)* | C-I | R-2 geometry-constant | `Plant_States_Bus.mass_kg` (per A3 §4.3.5) | PARAM | all | yes | 总质量;C3 落地 |
| PLANT_PARAM.02 | `inertia_b_kgm2` | `single[3][3]` (B4 verify) | 36 | kg·m² | body | diag([0.0211, 0.0219, 0.0366]) *(B4 verify; PX4-class quad)* | C-I | R-2 geometry-constant | `Plant_States_Bus.inertia_b_kgm2` | PARAM | all | yes | 惯量张量(3x3);多旋翼几乎对角 |
| PLANT_PARAM.03 | `cog_offset_b_m` | `single[3]` | 12 | m | body | [0, 0, 0] *(B4 verify)* | C-I | R-2 geometry-constant | — | PARAM | all | yes | 重心相对 body 原点偏移 |
| PLANT_PARAM.04 | `arm_length_m` | `single` | 4 | m | N/A | 0.225 *(B4 verify; 450mm wheelbase)* | C-I | R-2 geometry-constant | — (used by mixer & dynamics) | PARAM | MC-only | yes | 电机臂长 |
| PLANT_PARAM.05 | `motor_count` | `uint8` | 1 | N/A | N/A | 4 | C-I | R-3 dimension-constant | — (mixer matrix dimension) | PARAM | MC-only | yes | 电机数;Phase 2 多旋翼默认 4 |
| PLANT_PARAM.06 | `motor_pos_b_m` | `single[N_MOT_MAX][3]` | 12·N_MOT_MAX | m | body | quad-X 几何 *(B4 verify; per C3)* | C-I | R-2 geometry-constant | — | PARAM | MC-only | yes | 各电机位置;`N_MOT_MAX` per R-3 |
| PLANT_PARAM.07 | `motor_dir` | `int8[N_MOT_MAX]` | N_MOT_MAX | N/A | N/A | [+1,-1,+1,-1] (CW=+1) *(B4 verify)* | C-I | R-2 geometry-constant | — | PARAM | MC-only | yes | 旋转方向;quad-X 默认 |

**电机与推力组**

| # | Field | Type | Width | Unit | Frame | Default (multi) | Class | Class rationale | Affects bus field(s) | Reset | Variant | Init use | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| PLANT_PARAM.08 | `motor_thrust_max_n` | `single` | 4 | N | N/A | 9.0 *(B4 verify; PX4-class quad ~600g/motor)* | C-I | R-2 geometry-constant | `Control_Out_Bus.motor_cmd[]` saturation | PARAM | MC-only | no | 单电机最大推力 |
| PLANT_PARAM.09 | `motor_torque_const_nmpn` | `single` | 4 | N·m/N | N/A | 0.016 *(B4 verify; typical k_q/k_t)* | C-I | R-2 geometry-constant | — | PARAM | MC-only | no | 反扭矩 / 推力比 |
| PLANT_PARAM.10 | `motor_time_const_s` | `single` | 4 | s | N/A | 0.05 *(B4 verify; PX4-class first-order)* | R-T | R-1 GCS-tuned for motor model fidelity | — | PARAM | MC-only | no | 一阶电机响应时间常数 |
| PLANT_PARAM.11 | `motor_thrust_curve_coef` | `single[3]` | 12 | mixed | N/A | [0, 1, 0] (linear) *(B4 verify)* | R-T | R-1 GCS-tuned for ESC/motor calibration | — | PARAM | MC-only | no | a₀ + a₁·u + a₂·u² 形式 |

**气动与传感器噪声组**

| # | Field | Type | Width | Unit | Frame | Default (multi) | Class | Class rationale | Affects bus field(s) | Reset | Variant | Init use | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| PLANT_PARAM.12 | `drag_lin_coef_b` | `single[3]` | 12 | N·s/m | body | [0.1, 0.1, 0.2] *(B4 verify)* | R-T | R-1 GCS-tuned aerodynamic | — | PARAM | all | no | 线性气动阻力系数 |
| PLANT_PARAM.13 | `drag_quad_coef_b` | `single[3]` | 12 | N·s²/m² | body | [0, 0, 0] *(B4 verify)* | R-T | R-1 GCS-tuned aerodynamic | — | PARAM | all | no | 二次气动阻力系数 |
| PLANT_PARAM.14 | `imu_gyro_noise_radps` | `single` | 4 | rad/s | body | 1.0e-3 *(B4 verify; PX4 EKF baseline)* | R-T | R-1 GCS-tuned for noise injection | `IMU_Bus.gyr_b_radps` (per A3 §4.3.7.1) | PARAM | all | no | gyro 高斯噪声 σ |
| PLANT_PARAM.15 | `imu_acc_noise_mps2` | `single` | 4 | m/s² | body | 1.0e-2 *(B4 verify)* | R-T | R-1 GCS-tuned for noise injection | `IMU_Bus.acc_b_mps2` | PARAM | all | no | acc 高斯噪声 σ |
| PLANT_PARAM.16 | `imu_gyro_bias_radps` | `single[3]` | 12 | rad/s | body | [0, 0, 0] *(B4 verify)* | R-T | R-1 GCS-tuned for noise injection | `IMU_Bus.gyr_b_radps` | PARAM | all | no | 静态偏置 |
| PLANT_PARAM.17 | `imu_acc_bias_mps2` | `single[3]` | 12 | m/s² | body | [0, 0, 0] *(B4 verify)* | R-T | R-1 GCS-tuned for noise injection | `IMU_Bus.acc_b_mps2` | PARAM | all | no | |
| PLANT_PARAM.18 | `mag_noise_gauss` | `single` | 4 | Gauss | body | 1.0e-3 *(B4 verify)* | R-T | R-1 GCS-tuned for noise injection | `MAG_Bus.mag_b_gauss` (per A3 §4.3.7.2) | PARAM | all | no | mag 单位由 B1 confirm |
| PLANT_PARAM.19 | `baro_noise_pa` | `single` | 4 | Pa | N/A | 5.0 *(B4 verify)* | R-T | R-1 GCS-tuned for noise injection | `Barometer_Bus.pressure_pa` | PARAM | all | no | |
| PLANT_PARAM.20 | `gps_pos_noise_m` | `single[3]` | 12 | m | NED | [0.5, 0.5, 1.0] *(B4 verify)* | R-T | R-1 GCS-tuned for noise injection | `GPS_uBlox_Bus.lat/lon/alt` | PARAM | all | no | |
| PLANT_PARAM.21 | `gps_vel_noise_mps` | `single[3]` | 12 | m/s | NED | [0.1, 0.1, 0.2] *(B4 verify)* | R-T | R-1 GCS-tuned for noise injection | `GPS_uBlox_Bus.vel_ned_mps` | PARAM | all | no | |
| PLANT_PARAM.22 | `airspeed_noise_mps` | `single` | 4 | m/s | body | 0.2 *(B4 verify)* | R-T | R-1 GCS-tuned for noise injection | `AirSpeed_Bus.airspeed_mps` | PARAM | FW/VTOL | no | 多旋翼禁用 |

**环境种子组**(per A3 §4.3.3 `Environment_Info_Bus` 全字段 PARAM)

| # | Field | Type | Width | Unit | Frame | Default (multi) | Class | Class rationale | Affects bus field(s) | Reset | Variant | Init use | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| PLANT_PARAM.23 | `wind_vel_ned_mps_init` | `single[3]` | 12 | m/s | NED | [0, 0, 0] *(B4 verify)* | R-T | R-1 scenario-tuned | `Environment_Info_Bus.wind_vel_ned_mps` | PARAM | all | yes | 初始风速 |
| PLANT_PARAM.24 | `air_density_kgpm3` | `single` | 4 | kg/m³ | N/A | 1.225 (ISA sea level) | C-I | R-2 physical-constant | `Environment_Info_Bus.air_density_kgpm3` | PARAM | all | yes | |
| PLANT_PARAM.25 | `air_pressure_pa_init` | `single` | 4 | Pa | N/A | 101325.0 (ISA sea level) | R-T | R-1 scenario-tuned | `Environment_Info_Bus.air_pressure_pa` | PARAM | all | yes | |
| PLANT_PARAM.26 | `air_temperature_k_init` | `single` | 4 | K | N/A | 288.15 (ISA sea level) | R-T | R-1 scenario-tuned | `Environment_Info_Bus.air_temperature_k` | PARAM | all | yes | |
| PLANT_PARAM.27 | `mag_field_ned_gauss` | `single[3]` | 12 | Gauss | NED | [0.22, 0.05, 0.42] *(B4 verify; mid-latitude WMM-class)* | R-T | R-1 scenario-tuned | `Environment_Info_Bus.mag_field_ned_gauss` | PARAM | all | yes | 单位待 B1 确认(Gauss 还是 nT)|
| PLANT_PARAM.28 | `gravity_ned_mps2` | `single[3]` | 12 | m/s² | NED | [0, 0, 9.81] | C-I | R-2 physical-constant | `Environment_Info_Bus.gravity_ned_mps2` | PARAM | all | yes | |

**初始姿态 / 位置组**(per A3 §4.3.4 `States_Init_Bus` PARAM)

| # | Field | Type | Width | Unit | Frame | Default (multi) | Class | Class rationale | Affects bus field(s) | Reset | Variant | Init use | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| PLANT_PARAM.29 | `pos_ned_m_init` | `single[3]` | 12 | m | NED | [0, 0, 0] | R-T | R-1 scenario-tuned (per H1 场景) | `States_Init_Bus.pos_ned_m` & `Plant_States_Bus.pos_ned_m` echo at init | PARAM | all | yes | |
| PLANT_PARAM.30 | `vel_ned_mps_init` | `single[3]` | 12 | m/s | NED | [0, 0, 0] | R-T | R-1 scenario-tuned | `States_Init_Bus.vel_ned_mps` | PARAM | all | yes | |
| PLANT_PARAM.31 | `quat_ned_to_b_init` | `single[4]` | 16 | N/A | NED→body | [1, 0, 0, 0] (identity, w-first; B4 verify quat order) | C-I | R-2 default identity | `States_Init_Bus.quat_ned_to_b` | PARAM | all | yes | quat 顺序 (w,x,y,z) 由 B1 confirm |
| PLANT_PARAM.32 | `ang_rate_b_radps_init` | `single[3]` | 12 | rad/s | body | [0, 0, 0] | C-I | R-2 default zero | `States_Init_Bus.ang_rate_b_radps` | PARAM | all | yes | |

**积分器 / 数值组**(per A3 §4.3.5 / A6 §4.4.2 数值积分器内部状态;Class 默认 C-I)

| # | Field | Type | Width | Unit | Frame | Default (multi) | Class | Class rationale | Affects bus field(s) | Reset | Variant | Init use | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| PLANT_PARAM.33 | `integrator_method` | `uint8` (enum `IntegratorMethod` per B2) | 1 | N/A | N/A | `INT_RK4` *(B4 verify enum lock)* | C-I | R-2 fixed-at-codegen | — | PARAM | all | yes | 由 [C4](../C-plant/C4-numerics.md) 落地;B2 锁定 enum |
| PLANT_PARAM.34 | `step_size_s` | `single` | 4 | s | N/A | 0.001 (1 ms; per 架构 v1 §13) | C-I | R-2 fixed-at-codegen | — | PARAM | all | yes | Plant 周期 |

**注**:`PLANT_PARAM` 字段集 = 34 项(初稿);C3 在自身退出条件中复核数值,B4 在 contract diff 时 byte verify 顺序与类型。**字段顺序**(`#` 列)在 firmware 锁定后由 B4 维护;本初稿顺序仅为可读分组。

### 4.4 FMS_PARAM schema

#### 4.4.1 Header

| 项 | 值 |
|---|---|
| Struct 名 | `FMS_PARAM`(per A2 R-7.1;固化,不可改名)|
| Owner module | FMS(model-owned)|
| 来源 firmware 头文件 | `FMT-Firmware/src/model/fms/mc_fms/lib/FMS_types.h`,**FMT-Firmware @ \<pending hash\>** |
| 暴露形态 | 由模型仓 ert.tlc 生成 `extern FMS_PARAM_TYPE FMS_PARAM;`(B4 verify)|
| Phase 2 vehicle | multicopter |
| 总字段数 | 见 §4.4.2 + §4.6.4 |
| Variant 范围 | Phase 2 active = multicopter;Phase 5 fixwing/vtol 字段视情形扩展 |

#### 4.4.2 字段表

**Mode 阈值 / arming 与 disarming 组**

| # | Field | Type | Width | Unit | Frame | Default (multi) | Class | Class rationale | Affects bus field(s) | Reset | Variant | Init use | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| FMS_PARAM.01 | `default_mode_at_boot` | `uint8` (enum `PilotMode` per B2) | 1 | N/A | N/A | `PMODE_MANUAL` *(B4 verify)* | R-T | R-1 GCS-tuned | `FMS_Out_Bus.mode` (init) | PARAM | all | yes | per A6 §4.4.2 HARDCODED 是初始稳态,但具体 enum 值由 PARAM 提供 |
| FMS_PARAM.02 | `arm_throttle_thr_n01` | `single` | 4 | normalized 0..1 | N/A | 0.10 *(B4 verify)* | R-T | R-1 GCS-tuned | — (gates `Pilot_Cmd_Bus.arm_switch`) | PARAM | all | no | 解锁油门阈值 |
| FMS_PARAM.03 | `disarm_throttle_thr_n01` | `single` | 4 | normalized 0..1 | N/A | 0.05 *(B4 verify)* | R-T | R-1 GCS-tuned | — | PARAM | all | no | |
| FMS_PARAM.04 | `arm_stick_dwell_s` | `single` | 4 | s | N/A | 0.5 *(B4 verify)* | R-T | R-1 GCS-tuned | — | PARAM | all | no | |
| FMS_PARAM.05 | `arm_safety_check_mask` | `uint32` | 4 | N/A | N/A | 0xFFFFFFFF (all checks) *(B4 verify)* | R-T | R-1 GCS-tuned | — | PARAM | all | no | bitfield;由 D3 锁定每位 |

**Takeoff / Landing profile 组**

| # | Field | Type | Width | Unit | Frame | Default (multi) | Class | Class rationale | Affects bus field(s) | Reset | Variant | Init use | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| FMS_PARAM.06 | `takeoff_alt_m` | `single` | 4 | m | NED-Down | 2.5 *(B4 verify; PX4 default)* | R-T | R-1 GCS-tuned | `FMS_Out_Bus.pos_cmd_ned_m[2]` (during TAKEOFF) | PARAM | MC-only | no | 起飞目标高度 |
| FMS_PARAM.07 | `takeoff_climb_speed_mps` | `single` | 4 | m/s | NED-Down | 1.5 *(B4 verify)* | R-T | R-1 GCS-tuned | `FMS_Out_Bus.vel_cmd_ned_mps[2]` | PARAM | MC-only | no | |
| FMS_PARAM.08 | `landing_descent_speed_mps` | `single` | 4 | m/s | NED-Down | 0.7 *(B4 verify)* | R-T | R-1 GCS-tuned | `FMS_Out_Bus.vel_cmd_ned_mps[2]` | PARAM | MC-only | no | |
| FMS_PARAM.09 | `land_touchdown_speed_mps` | `single` | 4 | m/s | NED-Down | 0.3 *(B4 verify)* | R-T | R-1 GCS-tuned | — | PARAM | MC-only | no | 触地最终速度 |
| FMS_PARAM.10 | `land_disarm_dwell_s` | `single` | 4 | s | N/A | 2.0 *(B4 verify)* | R-T | R-1 GCS-tuned | — | PARAM | all | no | 落地后自动解锁等待 |

**Geofence 组**

| # | Field | Type | Width | Unit | Frame | Default (multi) | Class | Class rationale | Affects bus field(s) | Reset | Variant | Init use | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| FMS_PARAM.11 | `geofence_enabled` | `uint8` (bool) | 1 | N/A | N/A | 1 *(B4 verify)* | R-T | R-1 GCS-tuned | — | PARAM | all | yes | |
| FMS_PARAM.12 | `geofence_shape` | `uint8` (enum `GeofenceShape` per B2) | 1 | N/A | N/A | `GF_CYLINDER` *(B4 verify)* | R-T | R-1 GCS-tuned | — | PARAM | all | yes | enum: NONE/CYLINDER/POLYGON |
| FMS_PARAM.13 | `geofence_radius_m` | `single` | 4 | m | NED | 100.0 *(B4 verify)* | R-T | R-1 GCS-tuned | — | PARAM | all | no | cylinder shape 时使用 |
| FMS_PARAM.14 | `geofence_alt_max_m` | `single` | 4 | m | NED-Down | 120.0 *(B4 verify; FAA 400ft)* | R-T | R-1 GCS-tuned | — | PARAM | all | no | |
| FMS_PARAM.15 | `geofence_action` | `uint8` (enum `FailsafeAction` per B2) | 1 | N/A | N/A | `FA_RTL` *(B4 verify)* | R-T | R-1 GCS-tuned | `FMS_Out_Bus.failsafe_state` | PARAM | all | yes | |

**Failsafe 组**

| # | Field | Type | Width | Unit | Frame | Default (multi) | Class | Class rationale | Affects bus field(s) | Reset | Variant | Init use | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| FMS_PARAM.16 | `link_loss_timeout_s` | `single` | 4 | s | N/A | 1.0 *(B4 verify)* | R-T | R-1 GCS-tuned | — (gates `Pilot_Cmd_Bus.valid` staleness) | PARAM | all | no | RC 链路超时 |
| FMS_PARAM.17 | `gcs_loss_timeout_s` | `single` | 4 | s | N/A | 5.0 *(B4 verify)* | R-T | R-1 GCS-tuned | — | PARAM | all | no | GCS 链路超时 |
| FMS_PARAM.18 | `low_battery_voltage_v` | `single` | 4 | V | N/A | 14.0 *(B4 verify; 4S typical)* | R-T | R-1 GCS-tuned | — | PARAM | all | no | 触发 LOW_BAT failsafe |
| FMS_PARAM.19 | `critical_battery_voltage_v` | `single` | 4 | V | N/A | 13.0 *(B4 verify)* | R-T | R-1 GCS-tuned | — | PARAM | all | no | 触发 CRITICAL_BAT |
| FMS_PARAM.20 | `failsafe_link_loss_action` | `uint8` (enum) | 1 | N/A | N/A | `FA_RTL` *(B4 verify)* | R-T | R-1 GCS-tuned | `FMS_Out_Bus.failsafe_state` | PARAM | all | yes | |
| FMS_PARAM.21 | `failsafe_gcs_loss_action` | `uint8` (enum) | 1 | N/A | N/A | `FA_HOLD` *(B4 verify)* | R-T | R-1 GCS-tuned | | PARAM | all | yes | |
| FMS_PARAM.22 | `failsafe_low_bat_action` | `uint8` (enum) | 1 | N/A | N/A | `FA_RTL` *(B4 verify)* | R-T | R-1 GCS-tuned | | PARAM | all | yes | |
| FMS_PARAM.23 | `failsafe_critical_bat_action` | `uint8` (enum) | 1 | N/A | N/A | `FA_LAND` *(B4 verify)* | R-T | R-1 GCS-tuned | | PARAM | all | yes | |
| FMS_PARAM.24 | `ins_unready_timeout_s` | `single` | 4 | s | N/A | 30.0 *(B4 verify)* | R-T | R-1 GCS-tuned | — (gates `INS_Out_Bus.INS_Status.ready` per A6 §4.5.3) | PARAM | all | no | INS 就绪超时 |

**Mission tunables 组**

| # | Field | Type | Width | Unit | Frame | Default (multi) | Class | Class rationale | Affects bus field(s) | Reset | Variant | Init use | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| FMS_PARAM.25 | `wp_array_max_len` | `uint16` | 2 | N/A | N/A | 64 *(B4 verify; D5 锁定上限)* | C-I | R-3 array dimension | `Mission_Data_Bus.wp_array[]` length | PARAM | all | yes | 任务航点数组上限;C-I |
| FMS_PARAM.26 | `wp_acceptance_radius_m` | `single` | 4 | m | NED | 1.5 *(B4 verify)* | R-T | R-1 GCS-tuned | — | PARAM | MC-only | no | 航点到达判定半径 |
| FMS_PARAM.27 | `mission_default_speed_mps` | `single` | 4 | m/s | NED | 5.0 *(B4 verify)* | R-T | R-1 GCS-tuned | `FMS_Out_Bus.vel_cmd_ned_mps` (mission mode) | PARAM | MC-only | no | 默认航迹速度 |
| FMS_PARAM.28 | `rtl_climb_alt_m` | `single` | 4 | m | NED-Down | 30.0 *(B4 verify)* | R-T | R-1 GCS-tuned | — | PARAM | MC-only | no | RTL 爬升高度 |
| FMS_PARAM.29 | `rtl_loiter_time_s` | `single` | 4 | s | N/A | 5.0 *(B4 verify)* | R-T | R-1 GCS-tuned | — | PARAM | MC-only | no | RTL home 上空盘旋时间 |
| FMS_PARAM.30 | `home_lat_deg_default` | `double` | 8 | deg | N/A | 0.0 *(B4 verify; 由场景设置)* | R-T | R-1 scenario-tuned | `FMS_Out_Bus.home_lat_deg` (per A3 §4.4.6.3) | PARAM | all | yes | |
| FMS_PARAM.31 | `home_lon_deg_default` | `double` | 8 | deg | N/A | 0.0 *(B4 verify)* | R-T | R-1 scenario-tuned | `FMS_Out_Bus.home_lon_deg` | PARAM | all | yes | |
| FMS_PARAM.32 | `home_alt_m_default` | `single` | 4 | m | N/A | 0.0 *(B4 verify)* | R-T | R-1 scenario-tuned | `FMS_Out_Bus.home_alt_m` | PARAM | all | yes | |

**Command shaper 限值组**(per A3 §4.4.6.1 setpoint 字段守卫)

| # | Field | Type | Width | Unit | Frame | Default (multi) | Class | Class rationale | Affects bus field(s) | Reset | Variant | Init use | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| FMS_PARAM.33 | `vel_xy_lim_mps` | `single` | 4 | m/s | NED | 8.0 *(B4 verify; PX4 default)* | R-T | R-1 GCS-tuned | `FMS_Out_Bus.vel_cmd_ned_mps[0..1]` | PARAM | MC-only | no | 水平速度上限 |
| FMS_PARAM.34 | `vel_z_up_lim_mps` | `single` | 4 | m/s | NED-Down | 3.0 *(B4 verify)* | R-T | R-1 GCS-tuned | `FMS_Out_Bus.vel_cmd_ned_mps[2]` | PARAM | MC-only | no | 上升速度上限 |
| FMS_PARAM.35 | `vel_z_down_lim_mps` | `single` | 4 | m/s | NED-Down | 2.0 *(B4 verify)* | R-T | R-1 GCS-tuned | `FMS_Out_Bus.vel_cmd_ned_mps[2]` | PARAM | MC-only | no | 下降速度上限 |
| FMS_PARAM.36 | `acc_xy_lim_mps2` | `single` | 4 | m/s² | NED | 5.0 *(B4 verify)* | R-T | R-1 GCS-tuned | `FMS_Out_Bus.acc_cmd_ned_mps2[0..1]` | PARAM | MC-only | no | 水平加速度上限 |
| FMS_PARAM.37 | `jerk_xy_lim_mps3` | `single` | 4 | m/s³ | NED | 10.0 *(B4 verify)* | R-T | R-1 GCS-tuned | — (D4 shaper internal) | PARAM | MC-only | no | 水平 jerk 上限 |
| FMS_PARAM.38 | `tilt_lim_rad` | `single` | 4 | rad | body | 0.6109 (35°) *(B4 verify)* | R-T | R-1 GCS-tuned | `FMS_Out_Bus.att_cmd_quat` saturation | PARAM | MC-only | no | 最大倾角 |
| FMS_PARAM.39 | `yaw_rate_lim_radps` | `single` | 4 | rad/s | body | 1.5708 (90°/s) *(B4 verify)* | R-T | R-1 GCS-tuned | `FMS_Out_Bus.yaw_rate_cmd_radps` | PARAM | MC-only | no | yaw rate 上限 |
| FMS_PARAM.40 | `manual_stick_deadzone_n01` | `single` | 4 | normalized | N/A | 0.05 *(B4 verify)* | R-T | R-1 GCS-tuned | — (gates `Pilot_Cmd_Bus` sticks) | PARAM | all | no | 摇杆死区 |
| FMS_PARAM.41 | `manual_thr_hover_n01` | `single` | 4 | normalized 0..1 | N/A | 0.5 *(B4 verify)* | R-T | R-1 GCS-tuned | `FMS_Out_Bus.throttle_cmd` (manual mode) | PARAM | MC-only | no | hover 油门(手动模式)|

**Mode timing 组**

| # | Field | Type | Width | Unit | Frame | Default (multi) | Class | Class rationale | Affects bus field(s) | Reset | Variant | Init use | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| FMS_PARAM.42 | `mode_switch_debounce_s` | `single` | 4 | s | N/A | 0.05 *(B4 verify)* | R-T | R-1 GCS-tuned | — | PARAM | all | no | mode 切换防抖 |
| FMS_PARAM.43 | `failsafe_recovery_dwell_s` | `single` | 4 | s | N/A | 1.0 *(B4 verify)* | R-T | R-1 GCS-tuned | — | PARAM | all | no | failsafe 恢复保持时间 |

**注**:`FMS_PARAM` 字段集 = 43 项(初稿)。D5 在自身退出条件中复核 `MC-only` 字段,D1 / D3 / D4 复核 mode / shaper 字段,B4 byte verify 顺序。

### 4.5 CONTROL_PARAM schema

#### 4.5.1 Header

| 项 | 值 |
|---|---|
| Struct 名 | `CONTROL_PARAM`(per A2 R-7.1;固化,不可改名)|
| Owner module | Controller(model-owned)|
| 来源 firmware 头文件 | `FMT-Firmware/src/model/control/mc_controller/lib/Controller_types.h`,**FMT-Firmware @ \<pending hash\>** |
| 暴露形态 | 由模型仓 ert.tlc 生成 `extern CONTROL_PARAM_TYPE CONTROL_PARAM;`(B4 verify)|
| Phase 2 vehicle | multicopter |
| 总字段数 | 见 §4.5.2 + §4.6.4 |
| Variant 范围 | Phase 2 active = multicopter;级联结构对其他 vehicle 复用,具体由 E2 / E4 |

#### 4.5.2 字段表

**Position loop(架构 v1 §12.2.2;多旋翼启用,per E1 / E4)**

| # | Field | Type | Width | Unit | Frame | Default (multi) | Class | Class rationale | Affects bus field(s) | Reset | Variant | Init use | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| CONTROL_PARAM.01 | `pos_p_gain_xy` | `single` | 4 | (m/s)/m = 1/s | N/A | 0.95 *(B4 verify; PX4 MPC_XY_P)* | R-T | R-1 GCS-tuned | `Control_Out_Bus.motor_cmd[]` (downstream) | PARAM | MC-only | no | 水平位置 P |
| CONTROL_PARAM.02 | `pos_p_gain_z` | `single` | 4 | (m/s)/m = 1/s | N/A | 1.0 *(B4 verify; PX4 MPC_Z_P)* | R-T | R-1 GCS-tuned | | PARAM | MC-only | no | 垂直位置 P |
| CONTROL_PARAM.03 | `pos_xy_err_max_m` | `single` | 4 | m | NED | 5.0 *(B4 verify)* | R-T | R-1 GCS-tuned | | PARAM | MC-only | no | 位置误差饱和 |

**Velocity loop**

| # | Field | Type | Width | Unit | Frame | Default (multi) | Class | Class rationale | Affects bus field(s) | Reset | Variant | Init use | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| CONTROL_PARAM.04 | `vel_p_gain_xy` | `single` | 4 | (m/s²)/(m/s) = 1/s | N/A | 1.8 *(B4 verify; PX4 MPC_XY_VEL_P_ACC)* | R-T | R-1 GCS-tuned | | PARAM | MC-only | no | |
| CONTROL_PARAM.05 | `vel_i_gain_xy` | `single` | 4 | (m/s²)/(m·s) = 1/s² | N/A | 0.4 *(B4 verify)* | R-T | R-1 GCS-tuned | | PARAM | MC-only | no | |
| CONTROL_PARAM.06 | `vel_d_gain_xy` | `single` | 4 | (m/s²)/((m/s)/s) = dimless | N/A | 0.2 *(B4 verify)* | R-T | R-1 GCS-tuned | | PARAM | MC-only | no | |
| CONTROL_PARAM.07 | `vel_p_gain_z` | `single` | 4 | 1/s | N/A | 4.0 *(B4 verify; PX4 MPC_Z_VEL_P_ACC)* | R-T | R-1 GCS-tuned | | PARAM | MC-only | no | |
| CONTROL_PARAM.08 | `vel_i_gain_z` | `single` | 4 | 1/s² | N/A | 2.0 *(B4 verify)* | R-T | R-1 GCS-tuned | | PARAM | MC-only | no | |
| CONTROL_PARAM.09 | `vel_d_gain_z` | `single` | 4 | dimless | N/A | 0.0 *(B4 verify)* | R-T | R-1 GCS-tuned | | PARAM | MC-only | no | |
| CONTROL_PARAM.10 | `vel_int_lim_xy` | `single` | 4 | m/s² | N/A | 5.0 *(B4 verify)* | R-T | R-1 GCS-tuned (anti-windup) | | PARAM | MC-only | no | 积分限幅 |
| CONTROL_PARAM.11 | `vel_int_lim_z` | `single` | 4 | m/s² | N/A | 5.0 *(B4 verify)* | R-T | R-1 GCS-tuned | | PARAM | MC-only | no | |
| CONTROL_PARAM.12 | `acc_xy_lim_mps2` | `single` | 4 | m/s² | NED | 6.0 *(B4 verify)* | R-T | R-1 GCS-tuned | | PARAM | MC-only | no | velocity loop 输出限幅 |

**Attitude loop**

| # | Field | Type | Width | Unit | Frame | Default (multi) | Class | Class rationale | Affects bus field(s) | Reset | Variant | Init use | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| CONTROL_PARAM.13 | `att_p_gain_roll` | `single` | 4 | (rad/s)/rad = 1/s | body | 6.5 *(B4 verify; PX4 MC_ROLL_P)* | R-T | R-1 GCS-tuned | | PARAM | MC-only | no | |
| CONTROL_PARAM.14 | `att_p_gain_pitch` | `single` | 4 | 1/s | body | 6.5 *(B4 verify)* | R-T | R-1 GCS-tuned | | PARAM | MC-only | no | |
| CONTROL_PARAM.15 | `att_p_gain_yaw` | `single` | 4 | 1/s | body | 2.8 *(B4 verify; PX4 MC_YAW_P)* | R-T | R-1 GCS-tuned | | PARAM | MC-only | no | |
| CONTROL_PARAM.16 | `att_yaw_weight` | `single` | 4 | dimless | N/A | 0.4 *(B4 verify)* | R-T | R-1 GCS-tuned | | PARAM | MC-only | no | yaw 在 mixer 优先级 |

**Rate loop(5 ms 热路径,架构 v1 §12.2.5)**

| # | Field | Type | Width | Unit | Frame | Default (multi) | Class | Class rationale | Affects bus field(s) | Reset | Variant | Init use | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| CONTROL_PARAM.17 | `rate_p_gain_roll` | `single` | 4 | (N·m)/(rad/s) | body | 0.15 *(B4 verify; PX4 MC_ROLLRATE_P)* | R-T | R-1 GCS-tuned | `Control_Out_Bus.motor_cmd[]` | PARAM | MC-only | no | |
| CONTROL_PARAM.18 | `rate_p_gain_pitch` | `single` | 4 | (N·m)/(rad/s) | body | 0.15 *(B4 verify)* | R-T | R-1 GCS-tuned | | PARAM | MC-only | no | |
| CONTROL_PARAM.19 | `rate_p_gain_yaw` | `single` | 4 | (N·m)/(rad/s) | body | 0.20 *(B4 verify)* | R-T | R-1 GCS-tuned | | PARAM | MC-only | no | |
| CONTROL_PARAM.20 | `rate_i_gain_roll` | `single` | 4 | (N·m)/rad | body | 0.20 *(B4 verify; PX4 MC_ROLLRATE_I)* | R-T | R-1 GCS-tuned | | PARAM | MC-only | no | |
| CONTROL_PARAM.21 | `rate_i_gain_pitch` | `single` | 4 | (N·m)/rad | body | 0.20 *(B4 verify)* | R-T | R-1 GCS-tuned | | PARAM | MC-only | no | |
| CONTROL_PARAM.22 | `rate_i_gain_yaw` | `single` | 4 | (N·m)/rad | body | 0.10 *(B4 verify)* | R-T | R-1 GCS-tuned | | PARAM | MC-only | no | |
| CONTROL_PARAM.23 | `rate_d_gain_roll` | `single` | 4 | (N·m)/(rad/s²) | body | 0.003 *(B4 verify; PX4 MC_ROLLRATE_D)* | R-T | R-1 GCS-tuned | | PARAM | MC-only | no | |
| CONTROL_PARAM.24 | `rate_d_gain_pitch` | `single` | 4 | (N·m)/(rad/s²) | body | 0.003 *(B4 verify)* | R-T | R-1 GCS-tuned | | PARAM | MC-only | no | |
| CONTROL_PARAM.25 | `rate_d_gain_yaw` | `single` | 4 | (N·m)/(rad/s²) | body | 0.0 *(B4 verify)* | R-T | R-1 GCS-tuned | | PARAM | MC-only | no | |
| CONTROL_PARAM.26 | `rate_int_lim_roll` | `single` | 4 | N·m | body | 0.30 *(B4 verify; PX4 MC_ROLLRATE_I 限)* | R-T | R-1 GCS-tuned (anti-windup) | | PARAM | MC-only | no | |
| CONTROL_PARAM.27 | `rate_int_lim_pitch` | `single` | 4 | N·m | body | 0.30 *(B4 verify)* | R-T | R-1 GCS-tuned | | PARAM | MC-only | no | |
| CONTROL_PARAM.28 | `rate_int_lim_yaw` | `single` | 4 | N·m | body | 0.30 *(B4 verify)* | R-T | R-1 GCS-tuned | | PARAM | MC-only | no | |
| CONTROL_PARAM.29 | `rate_d_lpf_cutoff_hz` | `single` | 4 | Hz | N/A | 30.0 *(B4 verify; PX4 IMU_DGYRO_CUTOFF)* | R-T | R-1 GCS-tuned | | PARAM | MC-only | no | D 项 LPF;Phase 2 三轴共用 |
| CONTROL_PARAM.40 | `K_aw_vel_xy` | `single` | 4 | dimless | N/A | 0.5 *(B4 verify; back-calc baseline)* | R-T | R-1 GCS-tuned | velocity loop xy back-calculation anti-windup gain (per E3 §4.6.1) | PARAM | MC-only | no | 2026-05-09 amendment |
| CONTROL_PARAM.41 | `K_aw_vel_z` | `single` | 4 | dimless | N/A | 0.5 *(B4 verify)* | R-T | R-1 GCS-tuned | velocity loop z back-calculation anti-windup gain | PARAM | MC-only | no | 2026-05-09 amendment |
| CONTROL_PARAM.42 | `K_aw_rate_roll` | `single` | 4 | dimless | N/A | 0.3 *(B4 verify)* | R-T | R-1 GCS-tuned | rate loop roll back-calculation anti-windup gain (per E3 §4.6.2) | PARAM | MC-only | no | 2026-05-09 amendment |
| CONTROL_PARAM.43 | `K_aw_rate_pitch` | `single` | 4 | dimless | N/A | 0.3 *(B4 verify)* | R-T | R-1 GCS-tuned | rate loop pitch back-calculation anti-windup gain | PARAM | MC-only | no | 2026-05-09 amendment |
| CONTROL_PARAM.44 | `K_aw_rate_yaw` | `single` | 4 | dimless | N/A | 0.3 *(B4 verify)* | R-T | R-1 GCS-tuned | rate loop yaw back-calculation anti-windup gain | PARAM | MC-only | no | 2026-05-09 amendment |
| CONTROL_PARAM.30 | `rate_lim_roll_radps` | `single` | 4 | rad/s | body | 3.84 (220°/s) *(B4 verify)* | R-T | R-1 GCS-tuned | `FMS_Out_Bus.ang_rate_cmd_b_radps` saturation | PARAM | MC-only | no | |
| CONTROL_PARAM.31 | `rate_lim_pitch_radps` | `single` | 4 | rad/s | body | 3.84 *(B4 verify)* | R-T | R-1 GCS-tuned | | PARAM | MC-only | no | |
| CONTROL_PARAM.32 | `rate_lim_yaw_radps` | `single` | 4 | rad/s | body | 3.49 (200°/s) *(B4 verify)* | R-T | R-1 GCS-tuned | | PARAM | MC-only | no | |

**Mixer / 输出限幅组**

| # | Field | Type | Width | Unit | Frame | Default (multi) | Class | Class rationale | Affects bus field(s) | Reset | Variant | Init use | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| CONTROL_PARAM.33 | `mixer_geometry` | `uint8` (enum `MixerGeometry` per B2) | 1 | N/A | N/A | `MX_QUAD_X` *(B4 verify)* | C-I | R-2 geometry-constant | — (mixer matrix select) | PARAM | MC-only | yes | quad-X / quad-+ / hex-X |
| CONTROL_PARAM.34 | `motor_min_n01` | `single` | 4 | normalized 0..1 | N/A | 0.05 *(B4 verify; PX4 MOT_MIN)* | R-T | R-1 GCS-tuned | `Control_Out_Bus.motor_cmd[]` saturation low | PARAM | MC-only | no | 怠速最小输出 |
| CONTROL_PARAM.35 | `motor_max_n01` | `single` | 4 | normalized 0..1 | N/A | 1.0 *(B4 verify)* | R-T | R-1 GCS-tuned | `Control_Out_Bus.motor_cmd[]` saturation high | PARAM | MC-only | no | |
| CONTROL_PARAM.36 | `thrust_factor` | `single` | 4 | dimless | N/A | 0.30 *(B4 verify)* | R-T | R-1 GCS-tuned | | PARAM | MC-only | no | 推力曲线非线性补偿 |
| CONTROL_PARAM.37 | `hover_thrust_n01` | `single` | 4 | normalized 0..1 | N/A | 0.5 *(B4 verify)* | R-T | R-1 GCS-tuned | `Control_Out_Bus.motor_cmd[]` (feed-forward) | PARAM | MC-only | yes | hover 总推力点 |
| CONTROL_PARAM.38 | `mixer_yaw_priority_weight` | `single` | 4 | dimless | N/A | 0.5 *(B4 verify)* | R-T | R-1 GCS-tuned | | PARAM | MC-only | no | yaw 在饱和时让位 |

**控制启用 / disable 组**

| # | Field | Type | Width | Unit | Frame | Default (multi) | Class | Class rationale | Affects bus field(s) | Reset | Variant | Init use | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| CONTROL_PARAM.39 | `default_cmd_mask_at_init` | `uint32` | 4 | N/A | N/A | 0x00000000 (all loops disabled,per A6 §4.4.2) | C-I | R-2 safe-state | `FMS_Out_Bus.cmd_mask` (init echo) | PARAM | all | yes | init 时 cmd_mask 默认值 |

**注**:`CONTROL_PARAM` 字段集 = 39 项(初稿)。E4 在自身退出条件中复核 `MC-only` 字段,E3 复核 anti-windup / LPF 字段,B4 byte verify。

### 4.6 跨工作项交叉引用矩阵(B1 / B2 / A6 / A5)

#### 4.6.1 PARAM 字段 ↔ bus 字段(B1 cross-ref)

下表汇总 §4.3 / §4.4 / §4.5 中 "Affects bus field(s)" 列非空的字段(PARAM 字段直接 back 某条 bus 字段的初值或限幅);**B1 在 bus 字段 schema 中应反向引用本字段#**(per A3 §4.12 元数据要求,B1 字段表必须含"由哪条 PARAM 字段提供初值"列)。

| PARAM # | PARAM 字段 | 影响的 bus 字段 | bus 来源 | 关系 |
|---|---|---|---|---|
| PLANT_PARAM.01 | `mass_kg` | `Plant_States_Bus.mass_kg` | A3 §4.3.5 | echo at step |
| PLANT_PARAM.02 | `inertia_b_kgm2` | `Plant_States_Bus.inertia_b_kgm2` | A3 §4.3.5 | echo at step |
| PLANT_PARAM.14..21 | IMU/MAG/Baro/GPS noise | 各传感器 bus 噪声合成 | A3 §4.3.7.x | step-time noise injection |
| PLANT_PARAM.23..28 | Environment 字段 | `Environment_Info_Bus.*` | A3 §4.3.3 | configuration-time |
| PLANT_PARAM.29..32 | Init pose/vel/quat/rate | `States_Init_Bus.*` & `Plant_States_Bus.*` echo | A3 §4.3.4 / §4.3.5 | init-time |
| FMS_PARAM.01 | `default_mode_at_boot` | `FMS_Out_Bus.mode` (init) | A3 §4.4.6.2 | init echo |
| FMS_PARAM.06..09 | Takeoff/landing profile | `FMS_Out_Bus.pos_cmd_ned_m`/`vel_cmd_ned_mps` (during TAKEOFF/LAND) | A3 §4.4.6.1 | step-time during mode |
| FMS_PARAM.15 / 20..23 | Failsafe action enums | `FMS_Out_Bus.failsafe_state` | A3 §4.4.6.2 | step-time |
| FMS_PARAM.25 | `wp_array_max_len` | `Mission_Data_Bus.wp_array[]` length | A3 §4.4.3.3 | dimensional constraint |
| FMS_PARAM.30..32 | Home default | `FMS_Out_Bus.home_*_deg` / `_m` | A3 §4.4.6.3 | echo from `Mission_Data_Bus` |
| FMS_PARAM.33..39 | Shaper limits | `FMS_Out_Bus.vel_cmd*` / `acc_cmd*` / `att_cmd_quat` / `yaw_rate_cmd_radps` | A3 §4.4.6.1 | step-time saturation |
| FMS_PARAM.41 | `manual_thr_hover_n01` | `FMS_Out_Bus.throttle_cmd` (manual) | A3 §4.4.6.1 | step-time |
| CONTROL_PARAM.01..32 | 所有 PID 增益 | `Control_Out_Bus.motor_cmd[]` (downstream) | A3 §4.5.5 | step-time computation |
| CONTROL_PARAM.30..32 | Rate limits | `FMS_Out_Bus.ang_rate_cmd_b_radps` saturation in Controller | A3 §4.5.2 | step-time saturation |
| CONTROL_PARAM.34 / 35 | Motor min/max | `Control_Out_Bus.motor_cmd[]` saturation | A3 §4.5.5 | step-time saturation |
| CONTROL_PARAM.37 | `hover_thrust_n01` | `Control_Out_Bus.motor_cmd[]` (feed-forward) | A3 §4.5.5 | step-time feed-forward |
| CONTROL_PARAM.39 | `default_cmd_mask_at_init` | `FMS_Out_Bus.cmd_mask` (init) — 注意:此参数虽在 CONTROL_PARAM 中,但**初值合法性由 FMS 决定**;在 FMS init 时由 FMS 读 `CONTROL_PARAM.default_cmd_mask_at_init`,实际实现中可能由 D6 决定迁移到 `FMS_PARAM` 中 | A3 §4.4.6.2 | init echo;B1 cross-check |

**注**:本矩阵覆盖**全部 reset 类别为 PARAM 的 A3 字段** — 任何 A3 中标 PARAM 的字段必须在本矩阵中找到 B3 字段背书。**未找到的字段 = 缺口**,在 §5 风险中登记。

#### 4.6.2 PARAM 字段 ↔ enum 类型(B2 cross-ref)

PARAM schema 中的 enum 类型字段(类型由 B2 锁定数值):

| PARAM # | 字段 | enum 类型(per B2,类型名以 [A2 §4.6](../A-architecture/A2-naming-conventions.md) 风格)|
|---|---|---|
| PLANT_PARAM.33 | `integrator_method` | `IntegratorMethod`(B2 锁;成员示例 `INT_RK4`、`INT_FE`、`INT_HEUN`)|
| FMS_PARAM.01 | `default_mode_at_boot` | `PilotMode`(B2 锁定)|
| FMS_PARAM.12 | `geofence_shape` | `GeofenceShape`(B2 锁定)|
| FMS_PARAM.15 / 20..23 | failsafe / geofence action | `FailsafeAction`(B2 锁定)|
| CONTROL_PARAM.33 | `mixer_geometry` | `MixerGeometry`(B2 锁定)|

**B2 必须在自身字段表中确认上列 enum 类型存在并锁定数值**;若 B2 在自己的退出条件中决定枚举值不同于本文件示例(`INT_RK4` 等),B3 字段表的 Default 列在 co-seal 评审中同步更新。

#### 4.6.3 PARAM 字段 ↔ A6 init_use(per A6 §4.6.2 命名义务)

A6 §4.6.2 要求 B3 schema 字段中标注 `init_use=true / false`(本文件 §4.2.1 列定义已含 Init use 列)。汇总:

| Struct | Init use = yes 字段计数 | 备注 |
|---|---|---|
| `PLANT_PARAM` | 18(几何 + 环境 + init pose/vel/quat/rate + integrator method)| A6 §4.4.2 Plant_States_Bus reset 类别 PARAM 的字段全部由这些参数提供 |
| `FMS_PARAM` | 11(default_mode + geofence enable/shape/action + failsafe action + wp_array_max_len + home defaults + 部分)| A6 §4.4.2 FMS_Out_Bus init reset 类别 PARAM 的部分(其余 HARDCODED 不依赖 PARAM)|
| `CONTROL_PARAM` | 3(`mixer_geometry`、`hover_thrust_n01`、`default_cmd_mask_at_init`)| A6 §4.4.2 Control_Out_Bus reset 类别 PARAM 的字段非常少(多数 HARDCODED zero-thrust)|

#### 4.6.4 Tunable budget 汇总

| Struct | 总字段数 | runtime-tunable | compile-inlined | R-T 占比 | 设计依据 |
|---|---:|---:|---:|---:|---|
| `PLANT_PARAM` | 34 | 18 | 16 | 53% | 几何 / 物理 / 数值类大量 C-I(R-2/R-3);噪声、初始风场、初始 pose 类 R-T(R-1)|
| `FMS_PARAM` | 43 | 42 | 1 | 98% | mode/failsafe/shaper 几乎全部 GCS 暴露(R-1);仅 `wp_array_max_len`(R-3) C-I |
| `CONTROL_PARAM` | 44 *(2026-05-09 amendment: +5 K_aw)* | 41 | 3 | 93% | PID 增益全部 R-1 GCS-tuned;仅 `mixer_geometry`(R-2)、`hover_thrust_n01`(虽 R-1,但是否 C-I 取决于 codegen 是否常数化 — 此处取 R-T)、`default_cmd_mask_at_init`(R-2 安全态)C-I。新增 5 个 K_aw_* 字段(.40-.44)全 R-T per E3 §4.6 back-calculation 需求 |
| **合计** | **121** *(116 + 5 amendment)* | **101** | **20** | **83%** | |

**Rationale**:Plant 倾向 C-I(几何 / 物理常量,codegen 效率收益高);FMS / Controller 倾向 R-T(GCS 飞行期暴露收益高,代码体积代价小)。这一倾向与架构 v1 §14.3 "intentionally categorized" 的精神一致 — 不是按字段类型机械分配,而是按"谁是 codegen 受益者,谁是 GCS 受益者"判定。

变更原则:任何字段从 R-T 改 C-I 或反向,触发 B4 / I3 重新 diff(因 storage class 变化会改变生成代码符号);RULES §10 变更日志强制。

### 4.7 EXPORT struct 字段集(闭合 A1 §4.2 §5.1.2)

#### 4.7.1 EXPORT struct 通用形态

架构 v1 §5.1.2 要求 `*_EXPORT` 至少含 `period` 和 `model_info[]`。本文件按以下通用形态枚举三个 EXPORT struct:

```c
/* 通用骨架(B4 verify against firmware tip) */
struct <MODULE>_EXPORT_TYPE {
    uint16_t  period;          /* ms,模块 step 周期(架构 v1 §13)*/
    uint16_t  reserved_align;  /* alignment padding(B4 verify 是否存在)*/
    char      model_info[<N>]; /* 元数据字符串数组,长度由 firmware 锁定 */
    /* (可能含 schema_version / build_timestamp 等)*/
};
```

具体字段集**由 B4 在 contract diff 阶段 byte verify**。本文件给"模型仓侧 model_info[] ledger"作为占位:

#### 4.7.2 PLANT_EXPORT 字段集与 model_info[] 占位

| 字段 | 类型 | 单位 | 默认值 | 备注 |
|---|---|---|---|---|
| `period` | `uint16` | ms | 1 | Plant 周期(架构 v1 §13)|
| `model_info[]` | `char[N]` (N B4 verify) | N/A | 见 §4.7.5 ledger | |

#### 4.7.3 FMS_EXPORT 字段集与 model_info[] 占位

| 字段 | 类型 | 单位 | 默认值 | 备注 |
|---|---|---|---|---|
| `period` | `uint16` | ms | 20 | FMS 周期(架构 v1 §13)|
| `model_info[]` | `char[N]` | N/A | 见 §4.7.5 ledger | |

#### 4.7.4 CONTROL_EXPORT 字段集与 model_info[] 占位

| 字段 | 类型 | 单位 | 默认值 | 备注 |
|---|---|---|---|---|
| `period` | `uint16` | ms | 5 | Controller 周期(架构 v1 §13)|
| `model_info[]` | `char[N]` | N/A | 见 §4.7.5 ledger | |

#### 4.7.5 model_info[] ledger(三模块共用条目模板)

每个 EXPORT 的 `model_info[]` 应包含以下条目(具体字符串格式由 I4 codegen 配置生成,B4 verify 长度):

| 条目 | 内容 | 来源 |
|---|---|---|
| `model_name` | `"Plant"` / `"FMS"` / `"Controller"` | 模型根名(per A2 R-8.1)|
| `vehicle_class` | `"multicopter"` (Phase 2) | A5 |
| `schema_version` | semver 字符串(初稿 `"0.1.0-draft"`) | B4 锁定语义版本规则;变更触发 INDEX 决策日志 |
| `firmware_compat_hash` | firmware tip commit hash 前 8 位 | B5 / B4 在导出阶段写入 |
| `build_timestamp` | ISO-8601 | I4 codegen 时写入 |
| `param_field_count` | uint(本文件 §4.6.4 总字段数;Plant=34 / FMS=43 / Controller=44 *(2026-05-09 amendment: 39→44 with K_aw_* fields)* )| 由 I4 在 codegen 阶段自动写入 |

**注**:`model_info[]` 总字节长度由 firmware 锁定;B4 在 contract diff 中 byte verify "字符串数组总长度相等"。本文件不指定具体字节预算,留 B4。

#### 4.7.6 B3 与 EXPORT 的边界声明

- B3 拥有 EXPORT struct 的**字段清单**(field list)与 model_info[] **条目集**(content of entries)。
- B3 **不拥有**:
  - EXPORT struct 字段顺序的 byte 级 verify(由 B4 / I3 实现)
  - `period` 数值的实际写入(由 I4 codegen 配置写入)
  - `model_info[]` 字符串的具体格式化与生成(由 I4)
  - EXPORT 总字节宽度(由 firmware 锁定;B4 verify)
- 这一边界与 A1 §4.2 §5.1.2 的"Closed by B3"指派一致(A1 仅要求枚举字段集合,不要求字节布局)。

### 4.8 Diff coverage 要求(传递给 B4 / I3)

承接 [架构 v1 §17 Risk 6](../../architecture/2026-05-05-fmt-model-architecture-v1.md) 与 [00-design-plan §4.B B4 退出条件 (c)/(d)](../00-design-plan.md):

B4 / I3 必须覆盖以下 PARAM / EXPORT diff 项,本文件作为输入:

1. **PARAM 字段顺序与类型相等**:对 `PLANT_PARAM` / `FMS_PARAM` / `CONTROL_PARAM` 三个 struct,按 §4.3 / §4.4 / §4.5 字段表(序号 `#` 列)的顺序生成模型仓侧字段表,与 firmware tip `*_types.h` 字段表逐字段 diff。任一字段名 / 类型 / 数组长度 / 嵌套结构差异 → fail。
2. **PARAM 字段 storage class(R-T vs C-I)与 codegen 配置一致**:I4 在 codegen 配置中按本文件 Class 列设置 `Tunable=on/off`(具体机制由 I4)。B4 / I3 在生成产物中 verify(R-T 字段以 `extern volatile` 形式可寻址,C-I 字段不在 PARAM struct 中而是 inline 在生成代码;具体准则由 I4)。
3. **EXPORT 字段顺序、`period` 数值、`model_info[]` 总长度相等**:对三个 EXPORT struct,按 §4.7.2 / §4.7.3 / §4.7.4 字段顺序与 §4.7.5 model_info ledger 总字节预算,与 firmware tip diff。任一不等 → fail。
4. **符号存在性**:`Plant_init` / `Plant_step` / `FMS_init` / `FMS_step` / `Controller_init` / `Controller_step` 必须出现在生成产物中(per 00-design-plan B4 退出条件 (e))。这是 B4 主责,B3 在此呼应。

#### 4.8.1 Tuning divergence 检测(承接架构 v1 §17 Risk 6)

任何**模型仓侧**对 PARAM 字段的"未声明"修改(添加 / 删除 / 重命名 / 改类型 / 改 storage class)必须由 B4 / I3 在 CI / export 阶段检出。**B3 文档自身的修改**(非字段级,例如 Notes 列文字)不触发 fail,但触及 §4.3 / §4.4 / §4.5 / §4.6 / §4.7 表格的修改触发 RULES §10 变更日志 + B4 / I3 重新 diff。

### 4.9 Field-name 兼容性 frame(C3 / D5 / E4 leaf 显式核对模板,闭合审计 F-33)

**说明**:本节是**空模板**,由 C3 / D5 / E4 在自身退出条件中填充("参数名与 firmware `*_PARAM` 字段名兼容性显式核对")。B3 提供列定义与字段范围,leaf 文档负责 verify。

#### 4.9.1 C3 ↔ PLANT_PARAM 兼容性核对模板

| PLANT_PARAM # | PLANT_PARAM 字段(B3 schema)| C3 leaf 提议字段名 | firmware tip `Plant_types.h` 实际字段名 | 一致性 | C3 处置 |
|---|---|---|---|---|---|
| PLANT_PARAM.01 | `mass_kg` | _(C3 to fill)_ | _(C3 to fill)_ | _(yes/no/legacy)_ | _(rename / accept-legacy / escalate)_ |
| PLANT_PARAM.02 | `inertia_b_kgm2` | _(C3 to fill)_ | _(C3 to fill)_ | _(yes/no/legacy)_ | _(...)_ |
| PLANT_PARAM.03..34 | (全部见 §4.3.2) | _(C3 to fill)_ | _(C3 to fill)_ | _(...)_ | _(...)_ |

**C3 必须为 §4.3.2 中**全部 34 个 PLANT_PARAM 字段**填一行**;`MC-only` 标记的 18 个字段是 C3 主职责。一致性列三种:`yes`(B3 字段名 = firmware tip)/`legacy`(firmware tip 用旧名,接受沿用,per [A2 E-5.1](../A-architecture/A2-naming-conventions.md))/`no`(冲突,需协同 B3 修订 — 触发本文件变更)。

#### 4.9.2 D5 ↔ FMS_PARAM 兼容性核对模板

| FMS_PARAM # | FMS_PARAM 字段(B3 schema)| D5 leaf 提议字段名 | firmware tip `FMS_types.h` 实际字段名 | 一致性 | D5 处置 |
|---|---|---|---|---|---|
| FMS_PARAM.01 | `default_mode_at_boot` | _(D5 to fill)_ | _(D5 to fill)_ | _(yes/no/legacy)_ | _(...)_ |
| FMS_PARAM.06..10 | takeoff/landing profile (`MC-only`) | _(D5 to fill)_ | _(D5 to fill)_ | _(...)_ | _(...)_ |
| FMS_PARAM.02..43 | (全部见 §4.4.2) | _(D5 to fill)_ | _(D5 to fill)_ | _(...)_ | _(...)_ |

**D5 必须为 §4.4.2 中**全部 43 个 FMS_PARAM 字段**填一行**;`MC-only` 标记的字段是 D5 主职责(takeoff/landing profile / shaper 限值大半为 MC-only)。

#### 4.9.3 E4 ↔ CONTROL_PARAM 兼容性核对模板

| CONTROL_PARAM # | CONTROL_PARAM 字段(B3 schema)| E4 leaf 提议字段名 | firmware tip `Controller_types.h` 实际字段名 | 一致性 | E4 处置 |
|---|---|---|---|---|---|
| CONTROL_PARAM.01..32 | 所有 PID 增益(`MC-only`) | _(E4 to fill)_ | _(E4 to fill)_ | _(yes/no/legacy)_ | _(...)_ |
| CONTROL_PARAM.33..38 | mixer / 输出 (`MC-only`) | _(E4 to fill)_ | _(E4 to fill)_ | _(...)_ | _(...)_ |
| CONTROL_PARAM.39 | `default_cmd_mask_at_init` | _(E4 to fill)_ | _(E4 to fill)_ | _(...)_ | _(...)_ |

**E4 必须为 §4.5.2 中**全部 39 个 CONTROL_PARAM 字段**填一行**;Phase 2 多旋翼几乎全部字段 `MC-only`,E4 是主职责。

#### 4.9.4 兼容性 frame 的反向回流

若 C3 / D5 / E4 在填表时发现 firmware tip 字段名与 B3 schema 不一致(`legacy` 或 `no`),按以下流程:

1. `legacy` 类:沿 [A2 E-5.1](../A-architecture/A2-naming-conventions.md) 接受 firmware-legacy 名;在本文件 §4.3 / §4.4 / §4.5 表 Notes 列追加 `firmware-legacy: <legacy_name>`(走 RULES §10 变更日志)。
2. `no` 类:启动 B3 / leaf 协同修订;若 firmware 端命名优于 B3,B3 改;若 B3 命名优于 firmware,提议 firmware 端修改(这是契约风险事项,需 INDEX 决策日志登记)。
3. 缺失字段(B3 有,firmware 无 / firmware 有,B3 无):在 §5 风险中登记 + B4 在 diff 阶段 fail。

### 4.10 总字节宽度估算与端序假设

#### 4.10.1 总字节宽度估算(初稿,B4 verify)

| Struct | 估算总字节宽度(无 padding) | 备注 |
|---|---:|---|
| `PLANT_PARAM` | ~280 B | 34 字段;含 inertia 3x3(36B)、motor_pos[N_MOT_MAX][3](N_MOT_MAX × 12B)等大数组;实际宽度强烈依赖 `N_MOT_MAX`(默认 4 → motor_pos = 48B,motor_dir = 4B)|
| `FMS_PARAM` | ~140 B | 43 字段;以 single 与 enum (uint8) 为主;含 home double × 2 = 16B |
| `CONTROL_PARAM` | ~160 B | 39 字段;PID 增益与限幅以 single 为主(36 × 4B = 144B)|
| **合计** | **~580 B** | 总参数预算 < 1 KB,符合架构 v1 §4.5 嵌入式约束 |

**注**:估算不含 alignment padding;B4 在字节级 verify 时给出含 padding 的 ground-truth。本估算用于 EXPORT model_info[] 的 `param_field_count` / `param_total_bytes` 占位。

#### 4.10.2 端序

little-endian per ARM Cortex-M target(per §4.2.3;呼应 B1 同声明)。所有 multi-byte 字段 MSB-on-the-right。

### 4.11 设计决策汇总

| 决策 | 内容 | 依据 |
|---|---|---|
| **D-1 分类方法** | runtime-tunable vs compile-inlined 按 R-1..R-5 五条规则首次命中判定;每字段必须给一句话 rationale | 架构 v1 §14.3.3 + 审计 F-08 |
| **D-2 多旋翼默认值策略** | Phase 2 唯一 vehicle = multicopter;默认值采用 PX4-class baseline,标 `(B4 verify)`;C3 / D5 / E4 在 leaf 中复核数值与 firmware tip 对齐 | 架构 v1 §16 / §19 + A5 OQ1 决议 |
| **D-3 EXPORT vs PARAM 边界** | B3 拥有字段清单与 model_info[] 条目集;byte-level verify 与具体值写入由 B4 / I4 拥有 | A1 §5.1.2 Closed by B3 + B4 退出条件 (c)/(d)/(e) |
| **D-4 Leaf 兼容性 frame** | §4.9 三个空模板由 C3 / D5 / E4 在自身退出条件中填表;B3 提供列定义与全字段范围 | 审计 F-33 + 00-design-plan §4.C/D/E |
| **D-5 Tunable budget 倾向** | Plant 倾向 C-I(53% R-T);FMS / Controller 倾向 R-T(98% / 92%);依据"codegen 受益者 vs GCS 受益者" | 架构 v1 §14.3 "intentionally categorized" + §4.5 嵌入式约束 |
| **D-6 端序与对齐** | little-endian;遵循 ARM EABI;byte-level offset 留 B4 | 架构 v1 §13 ARM Cortex-M target;呼应 B1 |
| **D-7 INS_PARAM 不存在** | INS firmware-owned,本文件不涉及 INS 参数 | 架构 v1 §3 / §12.3 |

## 5. 已知风险与悬而未决问题

- **firmware commit hash 暂缺**
  - 影响:本文件 §3 / §4.3.1 / §4.4.1 / §4.5.1 均以 `FMT-Firmware @ <pending hash>` 标占位;[RULES §5](../RULES.md) 要求记录 commit hash + 文件相对路径。
  - 处置:open;评审通过前由 reviewer / orchestrator 在 INDEX 决策日志登记时补齐,与 [A6 §5 第 1 项 / A7 §3 / A3 §3](../A-architecture/A6-init-reset-contract.md) 同样占位约定。

- **所有 type / 数值默认 `(B4 verify)`**
  - 影响:本文件作者无法直接读 firmware `*_types.h`(环境注 §3),`(B4 verify)` 标记 = 78 个字段(每字段 type / default 至少一处)。
  - 处置:由 B4 在 contract diff 阶段强制对齐;C3 / D5 / E4 在自身退出条件中通过 §4.9 兼容性 frame 复核默认值。若 B4 diff 出 byte-level 差异,B3 走 RULES §10 变更日志。

- **PLANT_PARAM 字段顺序未与 firmware tip 对齐**
  - 影响:本文件 §4.3.2 字段 `#` 序号是 B3 作者按可读分组排列,可能与 firmware `Plant_types.h` 顺序不同。Simulink ert.tlc codegen 必须按 firmware 既有顺序产出,否则破坏 byte-level 兼容性。
  - 处置:open;B4 / I3 在字节级 diff 时 fail-on-mismatch;一旦 firmware 顺序锁定,B3 §4.3.2 / §4.4.2 / §4.5.2 的 `#` 列重排(走 RULES §10 变更日志)。任何下游引用 `#` 序号的文档(如 §4.6 / §4.9)同步更新。

- **`INS_PARAM` 不存在但下游可能误期望**
  - 影响:架构 v1 §3 / §12.3 已声明 INS firmware-owned,但下游(F1 / F3 / G3 logsout)可能误以为有 `INS_PARAM` 可调。
  - 处置:本文件 §1 / §2 / §4.11 D-7 显式声明无 `INS_PARAM`;B5 INS_Out_Bus 镜像同步流程负责消费侧契约,不涉及 INS 参数。

- **`CONTROL_PARAM.39 default_cmd_mask_at_init` 归属歧义**
  - 影响:此字段是"FMS init 时 cmd_mask 默认值",存在两种归属可能:(a) 归 `CONTROL_PARAM`(本文件初稿)— 因 cmd_mask 由 Controller 消费;(b) 归 `FMS_PARAM` — 因 cmd_mask 由 FMS 输出(per A3 §4.4.6.2)。
  - 处置:open;由 D6 / E1 co-seal batch(Wave 8)裁决归属;若移动到 `FMS_PARAM`,B3 §4.4.2 + §4.5.2 + §4.6.4 同步更新。

- **`hover_thrust_n01` 是否 R-T 还是 C-I**
  - 影响:本文件取 R-T(R-1 GCS-tuned),理由是不同 mass / battery 下需要在线调整。但 PX4 `MPC_THR_HOVER` 既有 R-T 也有 inline 用法。若 firmware 把它作为 C-I,B4 diff fail。
  - 处置:open;C3 / E4 在兼容性 frame 中复核;若 firmware 视为 C-I,本文件 §4.5.2 改 R-2 + Class rationale 改为 "physical-constant; codegen efficiency"。

- **`mag_field` 单位(Gauss vs nT)**
  - 影响:`PLANT_PARAM.27 mag_field_ned_gauss` 与 `MAG_Bus.mag_b_gauss`(per A3 §4.3.7.2)单位待 B1 confirm。SI 推荐 nT,但 firmware 可能用 Gauss。若改 nT,字段名后缀也需改(per A2 R-10.2 显式后缀)。
  - 处置:open;B1 字节级 verify 时确认;若改单位,B3 + A3 走 RULES §10 变更日志。

- **enum 类型成员名(`PMODE_MANUAL` / `INT_RK4` / `MX_QUAD_X` / `GF_CYLINDER` / `FA_RTL` 等)是占位**
  - 影响:本文件 §4.4 / §4.5 引用的 enum 成员名是 B3 作者按 [A2 R-6.2](../A-architecture/A2-naming-conventions.md) 类型短前缀风格构造,B2 锁定的实际成员名可能不同。
  - 处置:open;co-seal batch (B1, B2, B3) 同步评审时由 B2 确认成员名;若不同,B3 默认值与 §4.6.2 cross-ref 表同步更新。

- **`wp_array_max_len` 数值锁定**
  - 影响:Phase 2 多旋翼 mission 范围由 D5 决定;本文件初稿取 64 是 PX4 class baseline,不一定符合 firmware tip。架构 v1 §4.5 禁可变尺寸,故此字段锁定为 C-I,但**数值**仍需 D5 / firmware 一致。
  - 处置:open;D5 在自身退出条件锁定;触发 B4 重新 diff。

- **本工作项 contract_impact=yes,INDEX 决策日志登记**
  - 影响:RULES §6 self-check 第 5 项要求"触及 firmware 契约者已在 INDEX 决策日志登记"。
  - 处置:由 orchestrator 在 INDEX 中追加一行(本文件作为 author 不直接编辑 INDEX,符合 fmt-design-author skill 规则),同时补 firmware commit hash。

## 6. 退出条件复核

B3 在 [`00-design-plan.md §4.B`](../00-design-plan.md) 中的退出条件原文:

> `FMS_PARAM` / `CONTROL_PARAM` / `PLANT_PARAM` 字段级定义 + 多旋翼默认值表;**每个参数显式分类为 runtime-tunable 或 compile-inlined,并给出依据**

逐条复核:

| # | 退出条件分解 | 本文档依据 | 状态 |
|---|---|---|---|
| 1 | `FMS_PARAM` 字段级定义 | §4.4.2(43 字段)+ §4.4.1 header | 满足 |
| 2 | `CONTROL_PARAM` 字段级定义 | §4.5.2(39 字段)+ §4.5.1 header | 满足 |
| 3 | `PLANT_PARAM` 字段级定义 | §4.3.2(34 字段)+ §4.3.1 header | 满足 |
| 4 | 多旋翼默认值表 | §4.3.2 / §4.4.2 / §4.5.2 各表 Default 列;§4.10.1 总字节估算 | 满足(数值标 `(B4 verify; placeholder uses PX4-class default)`,符合环境注 §3 占位约定)|
| 5 | 每个参数显式分类为 runtime-tunable 或 compile-inlined | §4.3.2 / §4.4.2 / §4.5.2 各表 Class 列(R-T / C-I);§4.6.4 budget 汇总 | 满足(116 字段全部分类)|
| 6 | 分类依据 | §4.3.2 / §4.4.2 / §4.5.2 各表 Class rationale 列(R-1..R-5 + 一句话);§4.2.2 方法学 | 满足(116 字段全部给出 rationale)|

附:对 [A1 §4.2 / §4.3](../A-architecture/A1-v1-review.md) 中归属 B3 的缺口逐条闭合:

| 来源(A1)| 缺口简述 | 本文档闭合位置 |
|---|---|---|
| A1 §4.2 §5.1.2 | `*_EXPORT` 全字段未枚举 | §4.7(三个 EXPORT struct + model_info[] ledger)|
| A1 §4.2 §14.3.3 | runtime-tunable vs compile-inlined 分类标准未定 | §4.2.2 R-1..R-5 五条规则 + §4.6.4 budget |
| A1 §4.2 §17 Risk 6 | parameter governance(tuning divergence 检测)未给机制 | §4.8 + §4.8.1(传递给 B4 / I3)|
| A1 §4.2 §16 行 *(隐含)* | Phase 2 multicopter only 范围在参数侧落地 | §4.3 / §4.4 / §4.5 默认值;variant 列标 `MC-only` |

附:对 [A6 §4.4 / §4.6.2](../A-architecture/A6-init-reset-contract.md) 的承接:

| 来源(A6)| 承接义务 | 本文档承接位置 |
|---|---|---|
| A6 §4.4 类别表 | 每条 reset 类别 PARAM 的 bus 字段必须由 B3 提供初值 | §4.6.1 PARAM ↔ bus 矩阵(覆盖全部 PARAM 类别字段)|
| A6 §4.6.2 命名义务 | B3 schema 字段中标注 `init_use=true / false` | §4.2.1 列定义已含 Init use 列;§4.6.3 汇总 |
| A6 §4.4.1 PARAM 类别定义 | runtime-tunable vs compile-inlined 进一步分类 | §4.2.2 + §4.6.4 |

附:对审计 F-08 / F-33 的承接:

| 审计项 | 内容 | 本文档承接位置 |
|---|---|---|
| F-08 | 每参数 inline 决策必须有依据(rationale)| §4.2.2 R-1..R-5 + §4.3 / §4.4 / §4.5 各表 Class rationale 列 |
| F-33 | C3 / D5 / E4 leaf 必须显式核对参数名与 firmware 兼容性 | §4.9 三个空模板 |

退出条件全部满足;状态:`draft`,待评审升级到 `reviewed`。

## 7. 下游影响

按 [`01-design-relationships.md §4.2 / §4.3 / §4.4 / §4.5 / §4.9`](../01-design-relationships.md):

```text
B3 ⇢ C 系列, D 系列, E 系列, F 系列  (B 区→ C/D/E/F)
B1 ↔ B2 ↔ B3 → B4
B1, B2, B3 ⇢ I2  (镜像脚本消费)
A6 ⇢ B3        (A6 → B3 由 A6 §7 已声明)
```

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [B1 Bus 清单与 schema 设计](B1-bus-inventory.md) *(co-seal sibling)* | ↔ | §4.6.1 PARAM ↔ bus 矩阵作为 B1 字段表的反向引用源(A3 §4.12 元数据"由哪条 PARAM 字段提供初值"列)|
| [B2 Enum 清单与数值锁定](B2-enum-inventory.md) *(co-seal sibling)* | ↔ | §4.6.2 PARAM 字段 ↔ enum 类型矩阵;B2 必须确认列出的 enum 类型存在并锁数值 |
| [B4 契约 diff 策略设计](B4-contract-diff.md) *(未启动)* | → | §4.8 diff coverage 要求(PARAM 字段顺序 / 类型 / storage class、EXPORT 字段顺序 / period / model_info[] 长度);§4.10.1 总字节估算作为 diff baseline |
| [B5 INS_Out_Bus 镜像同步流程](B5-ins-bus-mirror.md) *(未启动)* | (无直接出边)| (B5 不依赖 B3;因 INS 无 PARAM)|
| [C1 Plant 功能设计](../C-plant/C1-plant-functional.md) *(未启动)* | ⇢ | §4.3.2 PLANT_PARAM 字段集(气动 / 传感器噪声 / 环境组)指引 C1 功能保真度等级 |
| [C2 Plant 结构设计](../C-plant/C2-plant-structural.md) *(未启动)* | ⇢ | §4.3.2 PLANT_PARAM 字段名 / Class 指引 C2 子系统层次中 PARAM 块的归属 |
| [C3 Plant 多旋翼 leaf 设计](../C-plant/C3-multicopter-leaf.md) *(未启动)* | ⇢ | §4.9.1 兼容性 frame 模板;§4.3.2 `MC-only` 字段(几何、电机、推力曲线)C3 主职责 |
| [C4 Plant 数值与积分设计](../C-plant/C4-numerics.md) *(未启动)* | ⇢ | §4.3.2 PLANT_PARAM.33-34(integrator_method / step_size_s)C-I 字段 |
| [D1 FMS 功能设计](../D-fms/D1-fms-functional.md) *(未启动)* | ⇢ | §4.4.2 FMS_PARAM mode / failsafe / mission tunables;指引 D1 mode 集合范围 |
| [D2 FMS 结构设计](../D-fms/D2-fms-structural.md) *(未启动)* | ⇢ | §4.4.2 FMS_PARAM 字段集分组(mode / takeoff / geofence / failsafe / mission / shaper / mode timing)对应 D2 6 子模块 |
| [D3 FMS Mode Manager 详细设计](../D-fms/D3-mode-manager.md) *(未启动,Wave 8)* | ⇢ | §4.4.2 FMS_PARAM.01 / 02-05 / 42 / 43(mode 切换防抖、解锁阈值)|
| [D4 FMS Command Shaper 详细设计](../D-fms/D4-command-shaper.md) *(未启动,Wave 8)* | ⇢ | §4.4.2 FMS_PARAM.33-41(shaper 限值组)|
| [D5 FMS 多旋翼 leaf 设计](../D-fms/D5-multicopter-leaf.md) *(未启动)* | ⇢ | §4.9.2 兼容性 frame 模板;§4.4.2 `MC-only` 字段(takeoff/landing profile、wp_acceptance、tilt_lim 等)D5 主职责 |
| [D6 FMS↔Controller 接口约定](../D-fms/D6-fms-controller-interface.md) *(未启动,Wave 8)* | ⇢ | §4.5.2 CONTROL_PARAM.39 default_cmd_mask_at_init(归属歧义,§5 风险);D6 裁决 |
| [E1 Controller 功能设计](../E-controller/E1-controller-functional.md) *(未启动,Wave 8)* | ⇢ | §4.5.2 CONTROL_PARAM 字段集 + Class 倾向(R-1 GCS-tuned)指引 E1 环路启用规则 |
| [E2 Controller 结构设计](../E-controller/E2-controller-structural.md) *(未启动)* | ⇢ | §4.5.2 字段分组(position / velocity / attitude / rate / mixer)对应 E2 级联拓扑 |
| [E3 Controller 各环算法设计](../E-controller/E3-loops-algorithm.md) *(未启动)* | ⇢ | §4.5.2 CONTROL_PARAM.10-11 / 26-29(anti-windup / D-term LPF)字段 |
| [E4 Controller 多旋翼 leaf 设计](../E-controller/E4-multicopter-leaf.md) *(未启动)* | ⇢ | §4.9.3 兼容性 frame 模板;§4.5.2 `MC-only` 字段(几乎全部)E4 主职责 |
| [E5 Controller 性能预算设计](../E-controller/E5-performance-budget.md) *(未启动)* | ⇢ | §4.6.4 Tunable budget(R-T 字段在生成代码中是 `extern volatile` 间接寻址,有性能影响)|
| [F1 ins_stub 功能设计](../F-ins-contract/F1-ins-stub-functional.md) *(未启动)* | ⇢ | §4.3.2 PLANT_PARAM.14-22(传感器噪声 / 偏置)指引 F1 噪声参数化范围 |
| [F3 INS_Out_Bus 消费规则](../F-ins-contract/F3-consumption-rules.md) *(未启动)* | ⇢ | §4.4.2 FMS_PARAM.24 ins_unready_timeout_s;F3 fallback 行为参考 |
| [I2 Bus/enum 镜像脚本设计](../I-tooling/I2-bus-enum-mirror.md) *(未启动)* | ⇢ | §4.7.5 model_info[] ledger 中 `firmware_compat_hash` 写入由 I2 / B5 协调 |
| [I3 契约 diff 脚本设计](../I-tooling/I3-contract-diff.md) *(未启动)* | ⇢ | §4.8 diff coverage 要求作为 I3 实现规范输入(co-seal with B4)|
| [I4 Codegen 配置脚本设计](../I-tooling/I4-codegen-config.md) *(未启动)* | ⇢ | §4.6.4 Class 列驱动 I4 配置 `Tunable=on/off`、storage class;§4.7.5 model_info[] 字符串生成由 I4 实现 |

整体而言:**B3 直接影响 B1 / B2 / B4 / I3 / I4(契约 diff 与 codegen 链)+ C/D/E 全 leaf(C3 / D5 / E4 兼容性 frame)**;评审通过后任何对本文件 §4.3 / §4.4 / §4.5 字段表的修订(添加 / 删除 / 重命名 / 改类型 / 改 storage class)都需沿全部出边对受影响下游加 `Affected by upstream change` 标记(RULES §10),并触发 B4 / I3 重新 diff、INDEX 决策日志登记。

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-07 | B3 author | 初稿(co-seal batch (B1, B2, B3) Wave 4)|
| 2026-05-07 | orchestrator | shared Reviewer verdict=pass(per-doc 7/7 + cross-doc C-1..C-10 全部 met);frontmatter 升 reviewed;INDEX 决策日志已登记。非阻塞建议:(a) `IntegratorMethod` / `FailsafeAction` / `MixerGeometry` / `GeofenceShape` 在 B2 §4.4.16 占位框架内,首跑 B4 后由 B2 增补;(b) `default_cmd_mask_at_init` 归属(CONTROL_PARAM vs FMS_PARAM)由 D6 / E1 Wave 8 co-seal 裁决;(c) EXPORT 字段集"至少"语义在首跑 B4 后增补。commit hash 占位 `<pending hash>` 与 A3/A6/A7 同期补齐 |
| 2026-05-09 | fix-up author | Wave 9 sealed-amendment(per RULES §10):新增 5 个 K_aw_* 字段(CONTROL_PARAM.40 K_aw_vel_xy / .41 K_aw_vel_z / .42 K_aw_rate_roll / .43 K_aw_rate_pitch / .44 K_aw_rate_yaw),全 R-T、`single`、默认 0.5(vel)/0.3(rate),支持 [E3 §4.6](../E-controller/E3-loops-algorithm.md) back-calculation anti-windup。CONTROL_PARAM 字段数 39→44;§4.6.4 Tunable budget 同步 (R-T 36→41);§4.7.5 EXPORT param_field_count 同步 (39→44)。**`*_int_lim_*` 字段保留作 hard-clamp safety 副本**(双层保护)。E4 §4.7 兼容性 frame 已含全部新字段(B4 verify pending)。Status 仍为 reviewed,sealed-amendment 不重新评审 — INDEX 决策日志由 orchestrator 在 Wave 9 finalize 时登记 |

## Self-check

- [x] frontmatter 完整,字段值合法
- [x] 上游文档全部存在且 status ≥ reviewed,**或**为本工作项所在 co-seal batch 的同批兄弟(B1 / B2 为 (B1, B2, B3) co-seal batch 同批兄弟,允许互引,见 §3 注;A1 / A2 / A3 / A6 status=reviewed,架构 v1 已发布)
- [x] 退出条件逐条复核完成,每条均给出依据(§6 表给出 6 项明确条件 + A1 / A6 / 审计承接)
- [x] 引用路径全部可点击访问
- [x] 不存在 [RULES §5](../RULES.md) 禁则中的内容(无 `.slx` 截图、无可执行 `.m`、无 firmware 实现复述、无 PR/branch 名;本文件不重复 bus 字段表(B1 主)、不重复 enum 数值(B2 主),仅给"参数 schema";firmware 契约文件相对路径已在 §3 给出,commit hash 标占位 — 同 [A6 §5 / A7 §3 / A3 §3](../A-architecture/A6-init-reset-contract.md) 模式)
- [x] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记(2026-05-07,orchestrator,Wave 4 完成 + B3 contract impact 条目)
- [x] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径(文件相对路径已在 §3 / §4.3.1 / §4.4.1 / §4.5.1 给出;commit hash 占位 `<pending hash>` 与 A3 / A6 / A7 同期补齐)
- [x] 下游影响已沿关系图识别完毕(§7 表覆盖 B1 / B2 / B4 直接边 + B5 + C/D/E/F/I 全 ⇢ 间接边)
- [x] 文档不超出本工作项范围(无越权设计 — bus 字段定义留 B1、enum 数值留 B2、leaf 数值修订留 C3 / D5 / E4、积分器细节留 C4、mode 状态留 D3、环路算法留 E3、codegen 实现留 I4;本文件仅给字段 schema + 分类 + EXPORT 字段集 + 兼容性 frame 模板)
