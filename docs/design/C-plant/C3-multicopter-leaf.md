---
work_item: C3
title: Plant 多旋翼 leaf 设计
upstream: [架构v1, A5, A6, B1, B3, C1, C2]
contract_impact: no
status: draft
authored_at: 2026-05-09
last_reviewed_at:
reviewer_verdict: none
---

# C3 Plant 多旋翼 leaf 设计

## 1. 目的

本文件为 Phase 2 多旋翼 vehicle 在 Plant 模块中的 **Layer C(leaf)** 填具体数值与算法常量,落地 [C2 §4.2.3](C2-plant-structural.md#423-layer-cleafmodelplantvehiclesmulticopter--c3-委托清单) 中 7 个 leaf 块(`MC_Geometry_Constants` / `MC_Mass_Inertia` / `MC_Motor_Model_Coeffs` / `MC_Allocation_Matrix` / `MC_Sensor_Placement_Offsets` / `MC_Init_Defaults` / `MC_Ground_Detect_Params`),并显式核对所有 PLANT_PARAM 字段名与 firmware tip `Plant_types.h` 字段名的兼容性(闭合 [审计 F-33](../_audit/2026-05-05-plan-audit.md))。

## 2. 范围

**在范围:**
- Phase 2 多旋翼默认拓扑 = quad-X(per [A5 §4.2 frozen actuator-count = 4](../A-architecture/A5-variant-strategy.md))
- 几何参数 leaf 数值(arm length / motor positions / motor directions / CG offset)
- 质量惯量 leaf 数值(total mass / 3×3 inertia tensor / off-diagonal = 0)
- 电机模型常数 leaf(thrust max / time constant / thrust polynomial / torque constant / motor failure injection point per [H4](../H-verification/H4-fault-catalog.md))
- 多旋翼分配矩阵(forward direction = motor cmd → body force / moment;Plant 侧使用)
- 初始条件 leaf 数值(pos / vel / quat / ang_rate;default ZERO + identity per [A6 §4.4.2](../A-architecture/A6-init-reset-contract.md))
- 多旋翼 sensor placement offset(VP-4;Phase 2 全零)
- §4.6 PLANT_PARAM 字段名兼容性核对(全 34 字段,闭合审计 F-33)

**不在范围(由其他工作项处理):**
- Plant 物理保真度等级 / 公式定义 — 由 [C1](C1-plant-functional.md)
- Plant 子系统层次 / 共享库块清单 / Variant 接入面 — 由 [C2](C2-plant-structural.md)
- 数值积分器 / 步长 / quat 单位化频率 / reset 时积分器内部状态 — 由 [C4](C4-numerics.md)
- Controller 侧混控逆向(force/moment cmd → motor cmd)— 由 [E4](../E-controller/E4-multicopter-leaf.md)
- 任何 bus / enum / PARAM 字段顺序与类型 schema — 由 [B1](../B-contracts/B1-bus-inventory.md) / [B2](../B-contracts/B2-enum-inventory.md) / [B3](../B-contracts/B3-parameter-schema.md)
- 故障注入参数化范围 / 接口签名 — 由 [H4](../H-verification/H4-fault-catalog.md)
- Phase 5 fixwing / VTOL leaf — 排除(per [00-design-plan §5](../00-design-plan.md))

## 3. 依赖

| 上游 | 引用位置 | 用途 |
|---|---|---|
| 架构 v1 §11 / §16 | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | Layer C vehicle leaf 原则 + Phase 2 multicopter slice |
| [A5 §4.2 / §4.3](../A-architecture/A5-variant-strategy.md) | Phase 2 frozen 维度(4 电机 / 单 active variant);A5 §4.3 Variant 预算合规 | 锁定 quad-X 是 Phase 2 唯一 active variant;`+` / hex 等仅留未来扩展占位 |
| [A6 §4.2 / §4.4.2](../A-architecture/A6-init-reset-contract.md) | `Plant_init(void)` 入口 + 初值类别(PARAM / HARDCODED / ZERO) | §4.5 初始条件类别归属 |
| [B1 §4.5 / §4.8](../B-contracts/B1-bus-inventory.md) | Plant 输出 bus(`Plant_States_Bus` / 5 sensor buses)字段级 schema | 字段名 / 类型严格引用,不在 C3 重定义 |
| [B3 §4.3 / §4.9.1](../B-contracts/B3-parameter-schema.md) | PLANT_PARAM 34 字段 schema + §4.9.1 C3 兼容性核对模板(空模板) | §4.6 把 §4.9.1 模板填满 |
| [C1 §4.5.3](C1-plant-functional.md) | 多旋翼 P3 推进/电机 L2 公式 + C1-OQ2 委托 | §4.3 数值落地 |
| [C2 §4.2.3 / §4.3.2](C2-plant-structural.md) | 7 leaf 块清单 + 5 VPs 接入点(V-AERO / V-PROP / V-INERTIA / V-SENSE-OFFSET / V-INIT-DEFAULTS) | §4.8 VP 落地 |

**Firmware 镜像引用**:`FMT-Firmware/src/model/plant/multicopter/lib/Plant_types.h`,**FMT-Firmware @ \<pending hash\>**(per [B3 §4.3.1 header](../B-contracts/B3-parameter-schema.md#431-header) 同步占位;首次 B4 跑通后由 [I3](../I-tooling/I3-contract-diff.md) verify hash)。

## 4. 设计内容

### 4.1 多旋翼几何(`MC_Geometry_Constants` leaf)

#### 4.1.1 拓扑选择

| 拓扑 | Phase 2 状态 | 备注 |
|---|---|---|
| **quad-X**(默认 + 唯一 active) | active | 4 电机均匀分布于 ±45° 方位;CW/CCW 交替;per [架构 v1 §16 / §19](../../architecture/2026-05-05-fmt-model-architecture-v1.md#19-recommended-first-implementation-target-after-this-document)|
| quad-+ | placeholder(未启用) | 同 4 电机但前后左右各 0°/90°/180°/270°;leaf 内通过 `motor_pos_b_m` 数据切换即可,**不**触发 [C2 §4.3.2 V-PROP](C2-plant-structural.md#432-plant-模块-variant-接入点枚举variant-subsystem-instances) 新 variant(per [C2 R-6](C2-plant-structural.md#5-已知风险与悬而未决问题)) |
| hex / octa | placeholder(未启用) | Phase 2 frozen `motor_count = 4`(per [A5 §4.2](../A-architecture/A5-variant-strategy.md));Phase 5 解冻 |
| coaxial / Y6 | not-considered | 不在 2025b roadmap |

**Phase 2 leaf 数值**全部基于 quad-X;`+` / hex 数值组占位由 Phase 5 leaf 扩展实现。

#### 4.1.2 quad-X 几何布局(NED body frame,FRD 约定 per [A2](../A-architecture/A2-naming-conventions.md))

```text
              x_b (forward)
                  ▲
          M0 (CW) │ M1 (CCW)
            \     │     /
             \    │    /
              \   │   /
               \  │  /
                \ │ /
   y_b ◄────────[CG]────────► (port wing not applicable)
                / │ \
               /  │  \
              /   │   \
             /    │    \
            /     │     \
          M3 (CCW)│ M2 (CW)
                  │
                 z_b (down)
```

电机编号约定(0..3,per [B1 `Control_Out_Bus.motor_cmd[]` index 顺序](../B-contracts/B1-bus-inventory.md));与 quad-X 的 `MC_Allocation_Matrix`(§4.4)一致:

| Motor index | body 位置 (m) | spin 方向(`motor_dir`)| 推力轴 | 备注 |
|---|---|---|---|---|
| `motor[0]` | `[+arm/√2, -arm/√2, 0]` | +1 (CW from above) | body -z(向下推↑机身) | 前左 |
| `motor[1]` | `[+arm/√2, +arm/√2, 0]` | -1 (CCW) | body -z | 前右 |
| `motor[2]` | `[-arm/√2, +arm/√2, 0]` | +1 (CW) | body -z | 后右 |
| `motor[3]` | `[-arm/√2, -arm/√2, 0]` | -1 (CCW) | body -z | 后左 |

`arm` = `arm_length_m`(`PLANT_PARAM.04`)= 单电机到 CG 在 xy 平面的欧氏距离。

`motor_pos_b_m`(`PLANT_PARAM.06`)= 上表第 2 列,排成 4×3 单精度数组(`single[4][3]`,Phase 2;Phase 5 数组上限由 firmware `N_MOT_MAX`)。

`motor_dir`(`PLANT_PARAM.07`)= `[+1, -1, +1, -1]`(per quad-X CW/CCW 交替);CW = +1 by convention(产生 +z body 反扭矩,机身受 -z 反作用力矩 → 自旋方向相反)。

**符号约定确认**:`motor_dir[k] * motor_torque_const_nmpn * thrust_per_motor_n[k]` 直接给出 body 系下 z 轴反扭矩;无需在 C3 引入第二符号。

#### 4.1.3 CG offset(`PLANT_PARAM.03 cog_offset_b_m`)

Phase 2 默认 `[0, 0, 0]` m(CG 与 body 原点重合;典型对称 quad 假设)。Phase 4+ 若引入挂载相机 / GPS 模块导致 CG 偏移,通过此字段表达,**不**改 Plant 子系统结构。

`MC_Geometry_Constants` leaf 输出端口:

```text
outputs:
  - arm_length_m            : single
  - motor_pos_b_m[4][3]     : single
  - motor_dir[4]            : int8
  - cog_offset_b_m[3]       : single
sources:
  - PLANT_PARAM.04 / .06 / .07 / .03
注入到 C2 的 V-PROP(S3)与 V-INERTIA(S4)
```

### 4.2 质量与惯量(`MC_Mass_Inertia` leaf)

#### 4.2.1 总质量

| 项 | PLANT_PARAM | Phase 2 默认 | 备注 |
|---|---|---|---|
| `mass_kg` | `PLANT_PARAM.01` | **1.5 kg** *(B4 verify;PX4-class quad baseline)* | 含电池 + 电机 + 机架;不含载荷;Phase 2 不模拟燃料/电池消耗,故仿真期间 constant |

#### 4.2.2 惯量张量(body 系,3×3,对角占主,per [B3 §4.3.2 PLANT_PARAM.02](../B-contracts/B3-parameter-schema.md#432-字段表))

```text
inertia_b_kgm2 =
  [ Ixx    -Ixy   -Ixz ]
  [-Iyx     Iyy   -Iyz ]
  [-Izx   -Izy    Izz  ]
```

| 元素 | Phase 2 默认 (kg·m²) | 备注 |
|---|---|---|
| Ixx | **0.0211** *(B4 verify;PX4-class quad)* | 滚转主惯量 |
| Iyy | **0.0219** *(B4 verify)* | 俯仰主惯量(对称 quad-X 通常 Ixx ≈ Iyy)|
| Izz | **0.0366** *(B4 verify)* | 偏航主惯量(quad 平板形,Izz > Ixx + Iyy 不必;但 Izz > Ixx, Izz > Iyy 是典型)|
| Ixy / Ixz / Iyz | **0** | 对称 quad-X 默认零;Phase 4+ 异形配置可解锁 |

来源 = `PLANT_PARAM.02 inertia_b_kgm2`(`single[3][3]` 对称矩阵)。

`MC_Mass_Inertia` leaf 输出 → 注入 [C2 §4.4.5 S4 Rigid Body 6-DoF](C2-plant-structural.md#445-s4-rigid-body-6-dof--kinematics) 经 V-INERTIA。

**注**:本节默认值与 [B3 §4.3.2 PLANT_PARAM.02](../B-contracts/B3-parameter-schema.md) 中给出的 `diag([0.0211, 0.0219, 0.0366])` 一致(C3 与 B3 数值同步,B4 verify 时统一对齐)。

### 4.3 电机模型(`MC_Motor_Model_Coeffs` leaf)

实现 [C1 §4.5.3 P3 推进 L2](C1-plant-functional.md#453-c1p3-推进电机l2-默认) 公式所需的常数与多旋翼专用参数。

#### 4.3.1 单电机推力公式(Phase 2 L2)

```pseudo
# (1) 一阶电机响应(motor cmd → 实际归一化推力)
u_actual[k] = first_order_filter(motor_cmd[k], tau = motor_time_const_s)

# (2) 推力多项式(单电机,c0/c1/c2 from PLANT_PARAM.11)
thrust_per_motor_n[k] = motor_thrust_max_n * (a0 + a1 * u_actual[k] + a2 * u_actual[k]^2)

# (3) 反扭矩(单电机,沿 body z)
torque_per_motor_nm[k] = motor_dir[k] * motor_torque_const_nmpn * thrust_per_motor_n[k]

# (4) H4 motor failure 注入(per DI-05;default scale = 1.0)
thrust_per_motor_n[k] *= motor_failure_scale[k]   # range [0, 1];H4 owns scale schema
```

#### 4.3.2 leaf 数值

| 参数 | PLANT_PARAM | 单位 | Phase 2 默认 | 备注 |
|---|---|---|---|---|
| `motor_thrust_max_n` | `PLANT_PARAM.08` | N | **9.0** *(B4 verify;PX4-class ~600 g/motor)* | 单电机最大推力;cmd=1 + a₀=0,a₁=1,a₂=0 时输出 = 9.0 N;4 motors total = 36 N → thrust/weight ≈ 36/(1.5·9.81) ≈ 2.45(健康 acro/sport quad)|
| `motor_torque_const_nmpn` | `PLANT_PARAM.09` | N·m/N | **0.016** *(B4 verify;典型 k_q/k_t)* | 反扭矩 / 推力比;高 KV 电机 ≈ 0.012,低 KV ≈ 0.020 |
| `motor_time_const_s` | `PLANT_PARAM.10` | s | **0.05** *(B4 verify;PX4-class first-order)* | 一阶电机响应时间常数;τ ≈ 50 ms = brushless ESC + propeller 综合 |
| `motor_thrust_curve_coef` | `PLANT_PARAM.11` | mixed | **[0.0, 1.0, 0.0]** *(B4 verify;linear)* | a₀ + a₁·u + a₂·u² 的 [a₀, a₁, a₂];Phase 2 默认线性 (a₁=1);Phase 4+ ESC calibration 可换成 [0, 0.4, 0.6](典型实测曲线) |

#### 4.3.3 单电机推力上下限

- **下限**(cmd=0):`thrust_per_motor_n = 9.0 * (0 + 1·0 + 0·0) = 0 N`(默认线性曲线下);电机最低推力 ≥ 0(命令 saturate 在 [0,1])。
- **上限**(cmd=1):`thrust_per_motor_n = 9.0 * (0 + 1·1 + 0·1) = 9.0 N`。
- **饱和块位置**:[C2 §4.4.4 S3 Propulsion sub: motor_cmd_saturation](C2-plant-structural.md#444-s3-propulsion--motor-model)(Layer A `Saturation_Symmetric` clamp [0,1] on motor_cmd);C3 不重新设计饱和块。

#### 4.3.4 motor_rpm 反算(`Plant_States_Bus.motor_rpm`,per [C1 §4.5.3](C1-plant-functional.md#453-c1p3-推进电机l2-默认))

```pseudo
# k_t = motor thrust constant ≈ motor_thrust_max_n / (max_rpm² * (2π/60)²)
# 由 leaf 派生(Phase 2 占位;实际值 by Phase 4 ESC calibration)
k_t_n_per_radps2 = motor_thrust_max_n / (omega_max_radps^2)
omega_radps[k]   = sqrt( max(thrust_per_motor_n[k], 0) / k_t_n_per_radps2 )
motor_rpm[k]     = omega_radps[k] * 60 / (2π)
```

`omega_max_radps` Phase 2 占位 = 1100 rad/s(≈ 10500 rpm,PX4-class brushless 典型上限);**不**作为独立 PLANT_PARAM 字段(Phase 2 内部派生),Phase 4+ 若需 GCS 可调,经 [B3](../B-contracts/B3-parameter-schema.md) 提交字段增补请求。

#### 4.3.5 motor failure 注入接口(per [H4](../H-verification/H4-fault-catalog.md) DI-05)

| 接口 | 类型 | 默认 | 备注 |
|---|---|---|---|
| `motor_failure_scale[4]` | `single[4]`,range [0, 1] | `[1, 1, 1, 1]`(无故障)| 0 = 该电机完全失效;0.5 = 推力衰减一半;**单电机 scale 独立**;H4 schema 具体 fault profile(瞬时 / 渐变)由 H4 owns |
| 注入位置 | C2 §4.4.4 S3 sub `motor_failure_mux` | — | mux input 1 = 1.0(default);input 2 = H4 driven;control by [DI-05](C2-plant-structural.md#47-扰动注入点的结构性-mux-安置per-c1-48-di-01di-06) |

### 4.4 分配矩阵(`MC_Allocation_Matrix` leaf,**forward direction**)

C3 落地的是 **forward** 分配矩阵(motor cmd / thrust → body force / moment),用于 Plant **propulsion → rigid body** 数据流;Controller 侧使用 **inverse**(force/moment cmd → motor cmd),由 [E4](../E-controller/E4-multicopter-leaf.md) 拥有。两者**不**共享数据(Plant 运行时不消费 inverse;Controller 运行时不消费 forward),但**必须**基于同一几何(arm_length / motor_pos_b / motor_dir)派生,以保持闭环一致性。

#### 4.4.1 quad-X 前向方程(per §4.1.2 几何 + §4.3.1 公式)

```text
设 T[k] = thrust_per_motor_n[k]         (单电机推力,N,沿 body -z)
设 r    = arm_length_m / sqrt(2)        (xy 投影距离,m)
设 c_q  = motor_torque_const_nmpn       (N·m/N)

前向 4×4 分配(quad-X,motor 编号 §4.1.2):
  Total thrust (body -z): F_z_b = -(T[0] + T[1] + T[2] + T[3])
  Roll moment  (body x):  M_x_b =  r * (T[1] + T[2] - T[0] - T[3])     # 右侧两电机 - 左侧两电机
  Pitch moment (body y):  M_y_b =  r * (T[0] + T[1] - T[2] - T[3])     # 前两电机 - 后两电机
  Yaw moment   (body z):  M_z_b =  c_q * (T[0] - T[1] + T[2] - T[3])   # CW(+1) 项 - CCW(-1) 项;与 motor_dir 一致
```

**符号校验**(右手 NED-FRD):
- 前两电机推力大 → `M_y_b > 0` → 机头上仰(俯仰俯仰角 θ 减少 ← 注意 body y 轴 = 右侧)→ 机身 nose-up 即 θ < 0(NED Z 向下);**与 [B1 / A2](../A-architecture/A2-naming-conventions.md) Euler 转角约定一致**(Z-Y-X intrinsic,roll/pitch/yaw 标准)。
- 右两电机推力大 → `M_x_b > 0` → 滚转 φ > 0 → 右翼下沉 → 与 NED-FRD 右手系一致。
- CW 电机(`motor_dir = +1`,`motor[0]` / `motor[2]`)推力大 → `M_z_b > 0` → 机头向右偏(yaw +)。

#### 4.4.2 矩阵化形式(供 leaf 视化或 codegen 使用)

```text
[F_z_b]     [ -1     -1     -1     -1 ]   [T[0]]
[M_x_b]  =  [ -r     +r     +r     -r ] * [T[1]]
[M_y_b]     [ +r     +r     -r     -r ]   [T[2]]
[M_z_b]     [+c_q   -c_q   +c_q   -c_q]   [T[3]]
```

leaf 输出 = 该 4×4 矩阵的数值实例化(由 `arm_length_m` / `motor_torque_const_nmpn` 在 init 时静态计算);**不**作为 `PLANT_PARAM` 独立字段(派生量;Phase 2 在 leaf 内静态计算,运行期不更新)。

#### 4.4.3 与 [E4 inverse](../E-controller/E4-multicopter-leaf.md)cross-reference(占位)

E4 owns 矩阵 `M_inv` 满足 `M_inv * [F_des, M_x_des, M_y_des, M_z_des]^T = T_des[k]`;由 §4.4.2 同一几何派生(伪逆 / 直接闭式;quad-X 4×4 可逆,直接 inverse 即可)。**C3 仅承诺 forward 方向数值落地;不**给出 E4 inverse 的实现(forward-cite E4)。

#### 4.4.4 拓扑切换(quad-X vs quad-+)

通过 §4.1.2 `motor_pos_b_m` 表更换为 quad-+ 几何(`[+arm, 0, 0]` / `[0, +arm, 0]` / `[-arm, 0, 0]` / `[0, -arm, 0]`)时,§4.4.1 公式中 `r → arm_length_m`,且 roll/pitch 行项简化为单电机贡献。leaf 内部以 `motor_pos_b_m` × `[0,0,-T[k]]` 的 cross product 实现(per [C1 §4.5.3](C1-plant-functional.md#453-c1p3-推进电机l2-默认)),**不**写死 quad-X 矩阵;矩阵展开形式仅用于人类阅读 + reviewer 校验。

### 4.5 初始条件(`MC_Init_Defaults` leaf)

实现 [A6 §4.4.2 Plant_States_Bus 初值类别](../A-architecture/A6-init-reset-contract.md) 多旋翼默认表(per [C1 §4.7.1](C1-plant-functional.md#471-plant_states_bus-字段类别映射));注入到 [C2 §4.6.2 Init Hub](C2-plant-structural.md#462-init-hubs0-结构) S0 经 V-INIT-DEFAULTS。

#### 4.5.1 PARAM-class 字段(数值由 PLANT_PARAM 提供)

| 字段 | PLANT_PARAM | Phase 2 默认 | A6 类别 | 注入到 |
|---|---|---|---|---|
| `pos_ned_m_init` | `PLANT_PARAM.29` | `[0, 0, 0]` m | PARAM | `States_Init_Bus.pos_ned_m` → `Plant_States_Bus.pos_ned_m` |
| `vel_ned_mps_init` | `PLANT_PARAM.30` | `[0, 0, 0]` m/s | PARAM | `States_Init_Bus.vel_ned_mps` → `Plant_States_Bus.vel_ned_mps` |
| `quat_ned_to_b_init` | `PLANT_PARAM.31` | `[1, 0, 0, 0]` (identity, w-first) | PARAM | `States_Init_Bus.quat_ned_to_b` |
| `ang_rate_b_radps_init` | `PLANT_PARAM.32` | `[0, 0, 0]` rad/s | PARAM | `States_Init_Bus.ang_rate_b_radps` |

**多旋翼语义说明**:Phase 2 默认 = 静止 + 水平 + 在原点(典型起飞前 ground state);H1 takeoff 场景 / H4 中空启动场景由 PLANT_PARAM 字段 override 这些默认。

#### 4.5.2 HARDCODED-class(leaf 内常量)

| 字段 | 默认 | 备注 |
|---|---|---|
| `Plant_States_Bus.on_ground` | `true` | per A6 §4.4.2 多旋翼默认在地面;首 step 内 S4 触地检测可即刻翻转 |
| `IMU_Bus.valid` / `MAG_Bus.valid` / `Barometer_Bus.valid` / `GPS_uBlox_Bus.valid` / `AirSpeed_Bus.valid` | `false` | per [C1 §4.6.4](C1-plant-functional.md#464-validity-flag-通用行为契约);warmup 后翻转 true |
| `GPS_uBlox_Bus.fix_type` | `UbloxFixType.NO_FIX` | per [B2](../B-contracts/B2-enum-inventory.md);GPS lock 状态机锁定 5s 后翻转 |

#### 4.5.3 ZERO-class

`Plant_States_Bus.acc_ned_mps2` / `acc_b_mps2` / `ang_acc_b_radps2` / `motor_rpm[]` / `timestamp` / `Extended_States_Bus.*`(全字段)/ 各传感器 noisy signals,均 ZERO at init,per [C1 §4.7.1–4.7.3](C1-plant-functional.md#471-plant_states_bus-字段类别映射)。leaf **不**重定义类别;仅引用。

#### 4.5.4 H1 / H4 场景 override

H1 verification 场景(per [H1](../H-verification/H1-scenario-catalog.md))与 H4 故障注入场景(per [H4](../H-verification/H4-fault-catalog.md))通过 harness 把 PLANT_PARAM 在仿真启动前 overwrite(per [C1 §4.7.5 Plant_init 契约](C1-plant-functional.md#475-plant_init-契约per-a6-423-plant-行) + [G4](../G-harness/G4-pilot-injection.md));**不**通过 leaf 内常量切换(避免 leaf 多版本)。leaf 仅持有 default;harness 持有 scenario-specific override。

### 4.6 PLANT_PARAM 字段名兼容性核对表(闭合审计 F-33)

填写 [B3 §4.9.1 模板](../B-contracts/B3-parameter-schema.md#491-c3--plant_param-兼容性核对模板),覆盖 [B3 §4.3.2](../B-contracts/B3-parameter-schema.md#432-字段表) 全 34 个 PLANT_PARAM 字段。

**填表说明**:
- "C3 提议字段名" = 本 leaf 文档主张的字段名(等同 B3 schema 名,**C3 不另起新名**)
- "firmware tip 字段名" = 待 [I3 contract diff](../I-tooling/I3-contract-diff.md) 首跑后 verify 实际 firmware `Plant_types.h` 的字段名;FMT-Firmware 未挂载,所有行标 `(B4 verify; expected <候选名>)`
- 一致性 = `TBD`(待 B4 verify);处置 = `align at first B4 run`(若发现 `legacy` / `no` 类,触发 [B3 §4.9.4 反向回流](../B-contracts/B3-parameter-schema.md#494-兼容性-frame-的反向回流))

**几何与质量惯量组**

| # | B3 字段 | C3 提议名 | firmware tip 名 | 一致性 | 处置 |
|---|---|---|---|---|---|
| .01 | `mass_kg` | `mass_kg` | (B4 verify; expected `mass_kg` or `Mass`) | TBD | align at first B4 run |
| .02 | `inertia_b_kgm2` | `inertia_b_kgm2` | (B4 verify; expected `inertia` or `J_b` or `inertiaTensor`) | TBD | align at first B4 run |
| .03 | `cog_offset_b_m` | `cog_offset_b_m` | (B4 verify; expected `cog_offset` or `cgOffset`) | TBD | align at first B4 run |
| .04 | `arm_length_m` | `arm_length_m` | (B4 verify; expected `arm_length` or `armLength` or `Arm`) | TBD | align at first B4 run |
| .05 | `motor_count` | `motor_count` | (B4 verify; expected `motor_count` or `nMotor` or `MotorCount`) | TBD | align at first B4 run |
| .06 | `motor_pos_b_m` | `motor_pos_b_m` | (B4 verify; expected `motor_pos` or `MotorPos` or `rotorPos`) | TBD | align at first B4 run |
| .07 | `motor_dir` | `motor_dir` | (B4 verify; expected `motor_dir` or `rotorDir` or `MotorDirection`) | TBD | align at first B4 run |

**电机与推力组**

| # | B3 字段 | C3 提议名 | firmware tip 名 | 一致性 | 处置 |
|---|---|---|---|---|---|
| .08 | `motor_thrust_max_n` | `motor_thrust_max_n` | (B4 verify; expected `motor_thrust_max` or `Tmax` or `MaxThrust`) | TBD | align at first B4 run |
| .09 | `motor_torque_const_nmpn` | `motor_torque_const_nmpn` | (B4 verify; expected `motor_torque_const` or `kQ` or `torqueCoef`) | TBD | align at first B4 run |
| .10 | `motor_time_const_s` | `motor_time_const_s` | (B4 verify; expected `motor_time_const` or `tauMotor` or `motorTau`) | TBD | align at first B4 run |
| .11 | `motor_thrust_curve_coef` | `motor_thrust_curve_coef` | (B4 verify; expected `thrust_curve` or `motorCurve` or `ThrustCoef`) | TBD | align at first B4 run |

**气动与传感器噪声组**

| # | B3 字段 | C3 提议名 | firmware tip 名 | 一致性 | 处置 |
|---|---|---|---|---|---|
| .12 | `drag_lin_coef_b` | `drag_lin_coef_b` | (B4 verify; expected `drag_lin_coef` or `dragLin` or `Cd_lin`) | TBD | align at first B4 run |
| .13 | `drag_quad_coef_b` | `drag_quad_coef_b` | (B4 verify; expected `drag_quad_coef` or `dragQuad` or `Cd_quad`) | TBD | align at first B4 run |
| .14 | `imu_gyro_noise_radps` | `imu_gyro_noise_radps` | (B4 verify; expected `gyro_noise` or `gyroSigma` or `IMU_Gyro_Noise`) | TBD | align at first B4 run |
| .15 | `imu_acc_noise_mps2` | `imu_acc_noise_mps2` | (B4 verify; expected `acc_noise` or `accSigma` or `IMU_Acc_Noise`) | TBD | align at first B4 run |
| .16 | `imu_gyro_bias_radps` | `imu_gyro_bias_radps` | (B4 verify; expected `gyro_bias` or `gyroBias`) | TBD | align at first B4 run |
| .17 | `imu_acc_bias_mps2` | `imu_acc_bias_mps2` | (B4 verify; expected `acc_bias` or `accBias`) | TBD | align at first B4 run |
| .18 | `mag_noise_gauss` | `mag_noise_gauss` | (B4 verify; expected `mag_noise` or `magSigma`;单位 Gauss vs nT 由 [B1](../B-contracts/B1-bus-inventory.md) confirm) | TBD | align at first B4 run |
| .19 | `baro_noise_pa` | `baro_noise_pa` | (B4 verify; expected `baro_noise` or `baroSigma`) | TBD | align at first B4 run |
| .20 | `gps_pos_noise_m` | `gps_pos_noise_m` | (B4 verify; expected `gps_pos_noise` or `gpsHorizSigma`) | TBD | align at first B4 run |
| .21 | `gps_vel_noise_mps` | `gps_vel_noise_mps` | (B4 verify; expected `gps_vel_noise` or `gpsVelSigma`) | TBD | align at first B4 run |
| .22 | `airspeed_noise_mps` | `airspeed_noise_mps` | (B4 verify; expected `airspeed_noise` or `vasSigma`;Phase 2 multicopter unused) | TBD | align at first B4 run |

**环境种子组**

| # | B3 字段 | C3 提议名 | firmware tip 名 | 一致性 | 处置 |
|---|---|---|---|---|---|
| .23 | `wind_vel_ned_mps_init` | `wind_vel_ned_mps_init` | (B4 verify; expected `wind_vel` or `windNED`) | TBD | align at first B4 run |
| .24 | `air_density_kgpm3` | `air_density_kgpm3` | (B4 verify; expected `air_density` or `rho`) | TBD | align at first B4 run |
| .25 | `air_pressure_pa_init` | `air_pressure_pa_init` | (B4 verify; expected `air_pressure_init` or `pBaroInit`) | TBD | align at first B4 run |
| .26 | `air_temperature_k_init` | `air_temperature_k_init` | (B4 verify; expected `air_temp_init` or `Tair`) | TBD | align at first B4 run |
| .27 | `mag_field_ned_gauss` | `mag_field_ned_gauss` | (B4 verify; expected `mag_field` or `magNED`;单位与 .18 同步)| TBD | align at first B4 run |
| .28 | `gravity_ned_mps2` | `gravity_ned_mps2` | (B4 verify; expected `gravity` or `gNED`) | TBD | align at first B4 run |

**初始姿态 / 位置组**

| # | B3 字段 | C3 提议名 | firmware tip 名 | 一致性 | 处置 |
|---|---|---|---|---|---|
| .29 | `pos_ned_m_init` | `pos_ned_m_init` | (B4 verify; expected `pos_init` or `posNEDInit`) | TBD | align at first B4 run |
| .30 | `vel_ned_mps_init` | `vel_ned_mps_init` | (B4 verify; expected `vel_init` or `velNEDInit`) | TBD | align at first B4 run |
| .31 | `quat_ned_to_b_init` | `quat_ned_to_b_init` | (B4 verify; expected `quat_init` or `attInit`;w-first vs x-last 由 [B1](../B-contracts/B1-bus-inventory.md) confirm) | TBD | align at first B4 run |
| .32 | `ang_rate_b_radps_init` | `ang_rate_b_radps_init` | (B4 verify; expected `ang_rate_init` or `omegaInit`) | TBD | align at first B4 run |

**积分器 / 数值组**

| # | B3 字段 | C3 提议名 | firmware tip 名 | 一致性 | 处置 |
|---|---|---|---|---|---|
| .33 | `integrator_method` | `integrator_method` | (B4 verify; expected `integrator_method` or `integMethod`;enum `IntegratorMethod` per [B2](../B-contracts/B2-enum-inventory.md)) | TBD | align at first B4 run |
| .34 | `step_size_s` | `step_size_s` | (B4 verify; expected `step_size` or `dtPlant` or `Ts`) | TBD | align at first B4 run |

**核对汇总**:
- 全部 **34** 字段已逐行登记(覆盖率 = 100%,**满足**任务要求 ≥ 30 字段)
- 所有 C3 提议名 **逐字等同** B3 schema 名(C3 不引入新命名;leaf 内变量名完全 echo PARAM 字段名)
- 一致性列全 `TBD`;处置统一 `align at first B4 run`,符合 [B3 §4.9.4](../B-contracts/B3-parameter-schema.md#494-兼容性-frame-的反向回流) 流程
- 18 个 `MC-only` 字段(.04 / .05 / .06 / .07 / .08 / .09 / .10 / .11 / .12 / .13 / .22 + 7 个共享但 C3 主职责的初值字段)是 C3 主职责;`all-vehicle` 字段(.01 / .02 / .03 / .14–.21 / .23–.34)C3 仅核对一致性,数值与 firmware 对齐由所有 vehicle leaf 共同遵循

### 4.7 Phase 2 默认值汇总(PX4-class baseline,B4 verify)

把 §4.1–§4.5 数值横向汇总,便于 reviewer 单次扫读;数值与 [B3 §4.3.2](../B-contracts/B3-parameter-schema.md#432-字段表) **逐字一致**。

| 类别 | 字段 | 默认 | 单位 |
|---|---|---|---|
| Mass | `mass_kg` | 1.5 | kg |
| Inertia | `inertia_b_kgm2[0][0]` (Ixx) | 0.0211 | kg·m² |
| Inertia | `inertia_b_kgm2[1][1]` (Iyy) | 0.0219 | kg·m² |
| Inertia | `inertia_b_kgm2[2][2]` (Izz) | 0.0366 | kg·m² |
| Inertia | off-diagonal | 0.0 | kg·m² |
| Geometry | `arm_length_m` | 0.225 | m |
| Geometry | `cog_offset_b_m` | [0, 0, 0] | m |
| Topology | `motor_count` | 4 | — |
| Topology | `motor_pos_b_m` | quad-X corners (§4.1.2) | m |
| Topology | `motor_dir` | [+1, -1, +1, -1] | — |
| Motor | `motor_thrust_max_n` | 9.0 | N |
| Motor | `motor_torque_const_nmpn` | 0.016 | N·m/N |
| Motor | `motor_time_const_s` | 0.05 | s |
| Motor | `motor_thrust_curve_coef` | [0, 1, 0] (linear) | mixed |
| Aero | `drag_lin_coef_b` | [0.1, 0.1, 0.2] | N·s/m |
| Aero | `drag_quad_coef_b` | [0, 0, 0] | N·s²/m² |
| Sensor σ | `imu_gyro_noise_radps` | 1.0e-3 | rad/s |
| Sensor σ | `imu_acc_noise_mps2` | 1.0e-2 | m/s² |
| Sensor σ | `mag_noise_gauss` | 1.0e-3 | Gauss |
| Sensor σ | `baro_noise_pa` | 5.0 | Pa |
| Sensor σ | `gps_pos_noise_m` | [0.5, 0.5, 1.0] | m |
| Sensor σ | `gps_vel_noise_mps` | [0.1, 0.1, 0.2] | m/s |
| Env | `air_density_kgpm3` | 1.225 (ISA SL) | kg/m³ |
| Env | `gravity_ned_mps2` | [0, 0, 9.81] | m/s² |
| Env | `wind_vel_ned_mps_init` | [0, 0, 0] | m/s |
| Init | `pos_ned_m_init` | [0, 0, 0] | m |
| Init | `vel_ned_mps_init` | [0, 0, 0] | m/s |
| Init | `quat_ned_to_b_init` | [1, 0, 0, 0] (identity, w-first) | — |
| Init | `ang_rate_b_radps_init` | [0, 0, 0] | rad/s |
| Numerics | `integrator_method` | `INT_RK4`(per [B2](../B-contracts/B2-enum-inventory.md);具体由 [C4](C4-numerics.md))| enum |
| Numerics | `step_size_s` | 0.001(1 ms;per 架构 v1 §13)| s |

**全部 标 `(B4 verify;PX4-class baseline)`**;首跑 [B4](../B-contracts/B4-contract-diff.md) / [I3](../I-tooling/I3-contract-diff.md) 后,数值漂移由 [B3 §10 changelog + INDEX 决策日志](../INDEX.md)登记。

### 4.8 Variant 接入点落地(per [C2 §4.3.2](C2-plant-structural.md#432-plant-模块-variant-接入点枚举variant-subsystem-instances) 5 VPs)

C2 已锁定 5 个 Variant 接入点;C3 在每个 VP 注入多旋翼 leaf 数值与算法。

| VP ID | C2 接入点 | C3 multicopter active variant 内容 | 本文档落地章节 |
|---|---|---|---|
| **VP-1**(V-PROP)| S3 Propulsion / Motor Model 内部 | `MC_Propulsion_4Rotor` = `MC_Geometry_Constants` + `MC_Motor_Model_Coeffs` + `MC_Allocation_Matrix`(forward direction)+ DI-05 motor failure mux | §4.1 / §4.3 / §4.4 |
| **VP-2**(V-AERO)| S2 Aerodynamics 内部 | `MC_Aero_L2_Drag` = consume `drag_lin_coef_b` (PLANT_PARAM.12) + `drag_quad_coef_b` (.13);C3 仅校对系数 default,公式由 [C1 §4.5.2](C1-plant-functional.md#452-c1p2-气动l2-默认) | §4.7(default 值)|
| **VP-3**(V-INERTIA)| S4 Rigid Body 6-DoF 内部 | `MC_Mass_Inertia` = `mass_kg` (.01) + `inertia_b_kgm2` (.02);几何由 V-PROP 提供 | §4.2 |
| **VP-4**(V-SENSE-OFFSET)| S5 Sensor Synthesis 内部 | `MC_Sensor_Placement_Offsets` Phase 2 全零 = IMU/MAG/Baro 在 IMU body 中心,**不**为 SIH 阶段升级预留 leaf 字段(per [C2 R-5](C2-plant-structural.md#5-已知风险与悬而未决问题);若 SIH 阶段需,经 [B3](../B-contracts/B3-parameter-schema.md) 字段增补)| §4.8.1(本节细节)|
| **VP-5**(V-INIT-DEFAULTS)| S0 Init Hub 内部 | `MC_Init_Defaults` = .29 / .30 / .31 / .32 PARAM-class echo + HARDCODED-class 常量(on_ground=true / valid=false / fix_type=NO_FIX)+ ZERO-class 派生量 | §4.5 |

#### 4.8.1 VP-4 sensor placement offset(详细)

`MC_Sensor_Placement_Offsets` leaf 输出端口(Phase 2 默认全零;Phase 4+ 经 PARAM 增补激活):

```text
outputs (Phase 2 all zero):
  - imu_offset_b_m[3]      : single  ([0, 0, 0])
  - imu_offset_quat_b[4]   : single  ([1, 0, 0, 0] identity)
  - mag_offset_b_m[3]      : single  ([0, 0, 0])
  - mag_offset_quat_b[4]   : single  ([1, 0, 0, 0])
  - baro_offset_b_m[3]     : single  ([0, 0, 0])
  - gps_offset_b_m[3]      : single  ([0, 0, 0])
sources (Phase 2):
  leaf 内置常量(**不**经 PLANT_PARAM;若需 GCS 调整,Phase 4+ 经 [B3](../B-contracts/B3-parameter-schema.md) 字段增补)
注入到 [C2 §4.4.6 S5](C2-plant-structural.md#446-s5-sensor-synthesis含-rb-07-downsamplers--详见-45) 的 sensor synthesis 内部 lever-arm 块(per [C1 §4.6.1 truth source](C1-plant-functional.md#461-各传感器测量模型表l3-default))
```

**理由**:Phase 2 多旋翼 MIL fidelity 不需要传感器 lever-arm 修正(PX4 EKF 假设 IMU 在 CG;偏差小于 noise σ × √dt);SIH 阶段(per 架构 v1 §15.3)真实 PCB 布局再启用。

### 4.9 Cross-references

- [C1](C1-plant-functional.md) — 多旋翼 P3 推进 / P2 气动 L2 公式;C3 提供常数
- [C2 §4.2.3 / §4.3.2](C2-plant-structural.md) — 7 leaf 块清单 + 5 VPs;C3 在每个 VP 注入数值
- [C4](C4-numerics.md) — `integrator_method` / `step_size_s` 落地;C3 仅承诺 default = `INT_RK4` / 0.001 s,具体由 C4
- [B1 §4.5 / §4.8](../B-contracts/B1-bus-inventory.md) — Plant 输出 bus 字段 schema;C3 不重定义,仅 echo
- [B3 §4.3 / §4.9.1](../B-contracts/B3-parameter-schema.md) — PLANT_PARAM 字段表 + 兼容性核对模板;§4.6 已填满
- [E4](../E-controller/E4-multicopter-leaf.md) — Controller 侧多旋翼混控 inverse;与 §4.4 forward 派生自同一几何(`arm_length_m` + `motor_pos_b_m` + `motor_dir`)
- [H1](../H-verification/H1-scenario-catalog.md) / [H4](../H-verification/H4-fault-catalog.md) — 验证场景与故障注入;§4.5.4 / §4.3.5 forward-cite
- [A5 §4.2 / §4.3](../A-architecture/A5-variant-strategy.md) — Phase 2 frozen 4 电机;§4.1.1 拓扑选择遵循
- [A6 §4.4.2](../A-architecture/A6-init-reset-contract.md) — Plant_init 多旋翼默认;§4.5 类别归属

## 5. 已知风险与悬而未决问题

- **R-1 PLANT_PARAM 默认值与 firmware tip 数值差异**(§4.7)
  - 影响:首次 [B4](../B-contracts/B4-contract-diff.md) / [I3](../I-tooling/I3-contract-diff.md) 跑通时,若 firmware tip 中 `PLANT_PARAM` 默认值与本文件 §4.7 不一致(例如 firmware 用 1.2 kg 而非 1.5 kg),触发 drift report
  - 处置:per [B3 §4.9.4](../B-contracts/B3-parameter-schema.md#494-兼容性-frame-的反向回流) firmware-canonical 默认胜出;C3 + B3 同步修订 §4.7 default;变更日志登记
- **R-2 quad-X vs quad-+ 在 leaf 内 PARAM 切换的运行时安全**(§4.1.1 + [C2 R-6](C2-plant-structural.md#5-已知风险与悬而未决问题))
  - 影响:`motor_pos_b_m` (.06) 在仿真启动前由 PARAM 配置;若中途 modify(不应,per [A6](../A-architecture/A6-init-reset-contract.md) PARAM 类约束),分配矩阵 §4.4.2 的派生值不会刷新
  - 处置:per [B3 §4.2.2 R-2 geometry-constant 分类](../B-contracts/B3-parameter-schema.md#422-分类方法runtime-tunable-vs-compile-inlined),`motor_pos_b_m` 是 `C-I`(compile-inlined)→ codegen 时 inline,无运行时 modify 通道;由 [I4 codegen config](../I-tooling/I4-codegen-config.md) 强制
- **R-3 `motor_thrust_curve_coef` 默认 [0, 1, 0] 与真实 ESC/电机非线性偏差**(§4.3.2)
  - 影响:H1 / H4 hover 场景 throttle 静态平衡可能偏离实测(典型电机 throttle ≈ 0.45 hover,但默认线性曲线下 hover throttle = (mass·g) / (4 · thrust_max) = 1.5·9.81 / 36 ≈ 0.41,接近但不完全匹配)
  - 处置:Phase 2 接受线性近似(MIL 不需要 throttle 实测匹配);Phase 4 ESC calibration 后由 [B3](../B-contracts/B3-parameter-schema.md) 改 [0, 0.4, 0.6] 等实测多项式
- **R-4 `omega_max_radps` 在 leaf 内派生,无 PARAM 字段暴露**(§4.3.4)
  - 影响:`motor_rpm` 反算的精度依赖 leaf 内 hardcoded 1100 rad/s;若 GCS 需调整(Phase 4+ ESC),需 B3 字段增补
  - 处置:Phase 2 标 leaf-private(类似 [C1-OQ10 sensor warmup](C1-plant-functional.md#49-open-issues--委托清单));Phase 4 必要时经 [B3](../B-contracts/B3-parameter-schema.md) 增补,触发 RULES §10 changelog
- **R-5 §4.4 forward 矩阵与 [E4](../E-controller/E4-multicopter-leaf.md) inverse 矩阵不同步漂移**(§4.4.3)
  - 影响:若 §4.1.2 几何变更但 E4 未同步,闭环力矩通道增益偏离
  - 处置:E4 同样从 PLANT_PARAM.04 / .06 / .07 派生 inverse(**不**独立持有几何);[B4](../B-contracts/B4-contract-diff.md) 字段顺序检查保证 PLANT_PARAM ≡ CONTROL_PARAM 在同一 firmware tip 同步;C3 / E4 各自负责自己方向的派生计算正确性;Wave 9 E4 与 C3 同 wave,但**不**co-seal(只是协同时点),非阻塞依赖
- **R-6 sensor placement offset Phase 2 全零的 SIH 升级负担**(§4.8.1)
  - 影响:SIH 阶段(per 架构 v1 §15.3)真实 PCB IMU 中心可能离 CG 数 cm,旋转动力学下 a_b ≠ a_CG_b
  - 处置:per [C2 R-5](C2-plant-structural.md#5-已知风险与悬而未决问题) + [C1-OQ10](C1-plant-functional.md#49-open-issues--委托清单);Phase 4 / SIH 启动前由 [H1 SIH 重跑列](../H-verification/H1-scenario-catalog.md) trigger 字段增补
- **R-7 `mag_field_ned_gauss` 单位 Gauss vs nT 与 firmware 不一致风险**(§4.6 .27 行)
  - 影响:若 firmware 用 nT,默认值 [0.22, 0.05, 0.42] Gauss × 1e5 = [22000, 5000, 42000] nT;数值 5 个数量级差异会被 [B4 byte-equality](../B-contracts/B4-contract-diff.md) 视为 fail
  - 处置:per [B1](../B-contracts/B1-bus-inventory.md) confirm 单位;若 firmware = nT,本文件 §4.7 + B3 §4.3.2 .27 行同步修订单位与数值;B4 verify

## 6. 退出条件复核

对照 [`00-design-plan.md` §4.C C3 行](../00-design-plan.md):**"几何、质量惯量、电机模型、分配矩阵、初始条件参数化方案;参数名与 firmware `PLANT_PARAM` 字段名兼容性显式核对"**(5 个 atomic 子条件 + 1 个审计闭合)。

| # | 退出条件原文(拆分)| 本文档依据 | 状态 |
|---|---|---|---|
| (a) | 几何 | §4.1 — quad-X 拓扑选择(§4.1.1)+ 4 电机布局表(§4.1.2)+ CG offset(§4.1.3)+ `MC_Geometry_Constants` leaf 输出端口;PLANT_PARAM .03 / .04 / .05 / .06 / .07 落地 | **满足** |
| (b) | 质量惯量 | §4.2 — 总质量(§4.2.1)+ 惯量张量 3×3(§4.2.2 含 Ixx / Iyy / Izz / off-diagonal=0)+ `MC_Mass_Inertia` leaf 注入 V-INERTIA;PLANT_PARAM .01 / .02 落地 | **满足** |
| (c) | 电机模型 | §4.3 — 推力公式(§4.3.1 一阶滤波 + 多项式 + 反扭矩 + DI-05 失效)+ leaf 数值表(§4.3.2)+ 推力上下限(§4.3.3)+ rpm 反算(§4.3.4)+ motor failure 接口(§4.3.5);PLANT_PARAM .08 / .09 / .10 / .11 落地 | **满足** |
| (d) | 分配矩阵 | §4.4 — forward 方向定义(§4.4.1 公式 + 符号校验)+ 矩阵化形式(§4.4.2)+ E4 inverse cross-ref(§4.4.3)+ 拓扑切换(§4.4.4);**显式声明 forward 方向,inverse 由 E4** | **满足** |
| (e) | 初始条件参数化方案 | §4.5 — PARAM-class(§4.5.1 .29 / .30 / .31 / .32)+ HARDCODED-class(§4.5.2 on_ground / valid / fix_type)+ ZERO-class(§4.5.3 acc / ang_acc / motor_rpm / timestamp / Extended_States_Bus)+ H1 / H4 override 路径(§4.5.4)| **满足** |
| (f) | **参数名与 firmware `PLANT_PARAM` 字段名兼容性显式核对**(闭合 [审计 F-33](../_audit/2026-05-05-plan-audit.md))| §4.6 — 全 34 字段逐行核对表(几何 7 + 电机 4 + 气动/噪声 11 + 环境 6 + 初值 4 + 数值 2);C3 提议名 = B3 schema 名(逐字一致);firmware tip 名 placeholder `(B4 verify; expected …)`;一致性 = TBD;处置 = `align at first B4 run`,符合 [B3 §4.9.4](../B-contracts/B3-parameter-schema.md#494-兼容性-frame-的反向回流) 流程 | **满足**(覆盖 ≥ 30 字段需求,实际 34/34 = 100%)|

辅助检查(非退出条件强制,但下游消费需要):

| # | 辅助 | 本文档依据 | 状态 |
|---|---|---|---|
| (g) | Phase 2 默认值汇总(便于 reviewer 单次扫读)| §4.7 全字段汇总表 | **满足** |
| (h) | C2 5 VPs 落地 | §4.8 表逐个 VP 落地;§4.8.1 VP-4 sensor offset 详细 | **满足** |
| (i) | A5 Phase 2 frozen 维度遵循 | §4.1.1 显式声明 quad-X = 唯一 active;`+` / hex / octa / coaxial 列 placeholder + 不启用 | **满足** |
| (j) | A6 init 类别遵循 | §4.5 三大类(PARAM / HARDCODED / ZERO)严格 echo C1 §4.7;leaf **不**重定义类别 | **满足** |
| (k) | RULES §5 内容禁则 | 无 .slx 截图 / 无可执行 .m 代码 / 无重复 firmware 实现细节 / 无重复 bus/enum/PARAM schema(全部引用 B 区单一来源)| **满足** |

## 7. 下游影响

按 [`01-design-relationships.md` §4.3](../01-design-relationships.md):

```text
C2 → C3                         (强前置;C3 在 leaf 块上填具体数值与算法)
C3 ⇢ C4                         (C3 §4.7 default integrator_method=INT_RK4 / step_size_s=0.001 是 C4 的输入)
C3 ⇢ E4                         (C3 §4.4 forward 矩阵几何 = E4 inverse 派生源)
C3 ⇢ H1 / H4                    (C3 §4.5.4 default 是 H1 场景的起始状态;C3 §4.3.5 motor_failure_scale 是 H4 接口签名)
C3 ⇢ B3                         (R-1 / R-4 / R-7 若触发字段增补/修订)
```

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [C4 Plant 数值与积分](C4-numerics.md) | ⇢ | §4.7 `integrator_method` / `step_size_s` 默认;§4.5 reset 时积分器内部状态契约 echo;§4.4.2 矩阵静态化策略(C4 决定是否在 leaf 还是 init script 计算)|
| [E4 Controller 多旋翼 leaf](../E-controller/E4-multicopter-leaf.md) | ⇢ | §4.1.2 几何 + §4.3.2 motor 常数(`arm_length_m` / `motor_pos_b_m` / `motor_dir` / `motor_torque_const_nmpn`)= E4 inverse 矩阵的派生源;§4.4 forward 数值供 E4 闭环验证 |
| [H1 验证场景目录](../H-verification/H1-scenario-catalog.md) | ⇢ | §4.5 default 起始状态 + §4.7 PX4-class baseline;H1 takeoff / hover / landing 场景以本默认为起点 |
| [H4 故障注入目录](../H-verification/H4-fault-catalog.md) | ⇢ | §4.3.5 `motor_failure_scale[4]` mux 接口签名 + DI-05 注入位置 cross-ref([C2 §4.7](C2-plant-structural.md#47-扰动注入点的结构性-mux-安置per-c1-48-di-01di-06))|
| [B3 PLANT_PARAM 增补](../B-contracts/B3-parameter-schema.md) | ⇢(条件性反馈)| §5 R-1 / R-4 / R-7 触发时,C3 向 B3 提交字段增补 / 修订请求(R-4 `omega_max_radps`;R-7 `mag_field_ned_gauss` 单位)|
| [G1 MIL 顶层](../G-harness/G1-mil-toplevel.md) | ⇢(经 C2 间接)| §4.5 default 起始状态;harness override 路径(§4.5.4)= G1 ↔ G4 配合 |

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-09 | C3 author | 初稿;闭合 [00-design-plan §4.C](../00-design-plan.md) C3 行 5 个 atomic 子条件 + 审计 F-33:(a) 几何(quad-X + arm/motor_pos/motor_dir/cog_offset);(b) 质量惯量(1.5 kg + diag([0.0211, 0.0219, 0.0366]) PX4-class);(c) 电机模型(thrust_max=9N / tau=0.05s / 线性多项式 / DI-05 motor failure);(d) 分配矩阵(quad-X forward 4×4 + 符号校验 + E4 inverse cross-ref);(e) 初始条件(PARAM/HARDCODED/ZERO 三类 echo C1 §4.7);(f) F-33 PLANT_PARAM 兼容性核对(全 34 字段逐行,覆盖率 100%,处置 = align at first B4 run)。Phase 2 默认值汇总表 §4.7 全部 标 `(B4 verify; PX4-class baseline)`。5 VPs 落地 §4.8 |

## Self-check

- [x] frontmatter 完整,字段值合法
- [x] 上游文档全部存在且 status ≥ reviewed:架构 v1(reviewed)、A5(reviewed 2026-05-05)、A6(reviewed 2026-05-05)、B1(reviewed 2026-05-07)、B3(reviewed 2026-05-07)、C1(reviewed 2026-05-08)、C2(reviewed 2026-05-08);无同 batch 兄弟
- [x] 退出条件逐条复核完成,每条均给出依据(§6 5 条原文 + F-33 闭合 + 5 条辅助 = 11 条全 "满足")
- [x] 引用路径全部可点击访问
- [x] 不存在 RULES §5 禁则中的内容(无 .slx 截图;无可执行 .m;无重复 firmware 实现细节;无重复 bus/enum/PARAM schema;镜像 firmware 引用已记录 commit hash 占位 `<pending hash>` + 文件路径 `FMT-Firmware/src/model/plant/multicopter/lib/Plant_types.h`)
- [N/A] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记 — 本文 `contract_impact=no`(数值与算法落地;契约由 B 区拥有)
- [N/A] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径 — 本文 `contract_impact=no`;§3 与 §4.6 已记录 firmware 引用占位以备 B4 verify
- [x] 下游影响已沿关系图识别完毕(§7:C4 / E4 / H1 / H4 / B3 / G1 全部覆盖)
- [x] 文档不超出本工作项范围(无越权设计):无 C1 物理保真度重定义;无 C2 子系统结构重定义;无 C4 积分器算法;无 E4 inverse 实现;无 B 区 schema 重定义;无 H1/H4 故障 schema)
