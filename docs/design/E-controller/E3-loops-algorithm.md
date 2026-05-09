---
work_item: E3
title: Controller 各环算法设计
upstream: [架构v1, A6, A7, B1, B2, B3, D6, E1, E2]
contract_impact: yes
status: reviewed
authored_at: 2026-05-09
last_reviewed_at: 2026-05-09
reviewer_verdict: pass
---

# E3 Controller 各环算法设计

## 1. 目的

锁定 Controller 五个级联环 (L-01 Position / L-02 Velocity / L-03 Attitude / L-04 Angular rate / L-05 Mixer) 的**控制律数学定义、前馈接入、anti-windup 算法、滤波器设计与离散化策略**,在 [E2](E2-controller-structural.md) 提供的 `Controller_Cascade_Loop_Shell` 骨架槽位中填入算法,使下游 [E4](E4-multicopter-leaf.md) 多旋翼 leaf 能够注入数值增益、[E5](E5-performance-budget.md) 性能预算可基于本算法规模评估 5 ms 周期内执行成本。

## 2. 范围

**在范围:**
- L-01 / L-02 / L-03 / L-04 四个环的控制律数学方程(P / PI / PID 选型 + 误差量定义)
- 各环前馈接入策略(`vel_ff`、`acc_ff`、重力前馈)与来源字段(per E1 cmd_mask 路由)
- L-02 / L-04 两个含积分器环的 anti-windup 算法(back-calculation)
- 测量滤波器(velocity LPF / 角速度 LPF / D 项 LPF)的截止频率与离散化方法
- 姿态环的 quaternion 误差解算公式(含最短路径符号修正)
- 各环 init / reset 后的算法侧稳态(integrator/filter state = 0)
- 算法层抗噪与饱和复位行为约定(交付 E4 / E5 的输入)

**不在范围(由其他工作项处理):**
- 级联拓扑、子系统编排、共享库块清单 — 由 [E2](E2-controller-structural.md) 处理
- cmd_mask 裁剪规则 / 环路 enable 矩阵 / bypass 语义 — 由 [E1](E1-controller-functional.md) 处理
- 多旋翼分配矩阵、电机推力曲线、L-05 Mixer 内部 — 由 [E4](E4-multicopter-leaf.md) 处理
- 数值增益(K_p / K_i / K_d / K_aw 具体值)与饱和限幅具体值 — 由 [E4](E4-multicopter-leaf.md) 在 `CONTROL_PARAM` 中提供
- 5 ms 周期内执行预算、热点禁用项、查找表 / 定点策略 — 由 [E5](E5-performance-budget.md) 处理
- bus 字段 / enum / 参数 struct schema — 由 [B1](../B-contracts/B1-bus-inventory.md) / [B2](../B-contracts/B2-enum-inventory.md) / [B3](../B-contracts/B3-parameter-schema.md) 处理
- L-05 (Mixer) 算法本身不在 E3 — E3 仅声明它消费 L-04 输出,不定义控制律(纯线性分配,无律可言)

## 3. 依赖

| 上游 | 引用位置 | 用途 |
|---|---|---|
| 架构 v1 §7.2 | [架构 v1](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | Controller 顶层职责:cascaded loops + feed-forward + anti-windup + mixer |
| 架构 v1 §12.2 | [架构 v1](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | 5 stage 内部 stack(Setpoint Decode / Position / Velocity / Attitude / Rate / Mixer / Output Assembler);"PI/PID + feed-forward acceleration/thrust";"quaternion-aware attitude error";"anti-windup and LPF D-term discipline" |
| 架构 v1 §13 | [架构 v1](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | Controller single-rate 5 ms / 200 Hz;NO variable-step;离散化必须显式 |
| A6 §4.2.3 + §4.4.2 | [A6 init/reset 契约](../A-architecture/A6-init-reset-contract.md) | Controller_init 各环 integrator / antiwindup / filter state = 0;reset 后稳态 = init 后稳态;"reset 后无 transient" |
| A7 | [A7 时间约定](../A-architecture/A7-time-conventions.md) | dt 来源:基于 200 Hz fixed-step,不基于 timestamp;离散滤波器使用固定 Ts = 5 ms |
| B1 §4.7 | [B1 Bus 清单](../B-contracts/B1-bus-inventory.md) | `att_cmd_quat[4]` vs `att_cmd_euler_rad[3]` 二选一(per Wave 4 非阻塞建议;E3 锁定 quat 路径为主、euler 仅初始化转换) |
| B2 §4.4.14 | [B2 Enum 清单](../B-contracts/B2-enum-inventory.md) | cmd_mask 位号(POSITION_LOOP=0 / VELOCITY_LOOP=1 / ACCELERATION_LOOP=2 / ATTITUDE_LOOP=3 / RATE_LOOP=4 / YAW_LOOP=5 / YAW_RATE_LOOP=6 / THROTTLE_PASSTHROUGH=7);算法仅消费位号语义,不定义 |
| B3 §4.6.4 | [B3 Parameter schema](../B-contracts/B3-parameter-schema.md) | `CONTROL_PARAM` 39 字段:增益子集 `K_p_pos_*` / `K_p_vel_*` / `K_i_vel_*` / `K_p_att_*` / `K_p_rate_*` / `K_i_rate_*` / `K_d_rate_*` / `K_aw_*` / 滤波器截止 / saturation 限幅(具体字段名属于 E4 leaf 兼容性核对) |
| D6 §4.3 | [D6 FMS-Controller 接口](../D-fms/D6-fms-controller-interface.md) | cmd_mask 位优先级 + 前馈位与主位的 co-set 关系(POS+VEL 共置位 = POS 主 + VEL ff;POS/VEL+ACC = ACC ff)— E3 据此决定前馈接入策略 |
| E1 §4.1 + §4.3 | [E1 Controller 功能设计](E1-controller-functional.md) | 5 环路总表 + 各环 setpoint / measurement 来源 / 内部 wire 名称(`vel_sp_inner` / `ang_rate_sp_inner`)/ bypass 语义;特别是 §4.3.5 Wave 8 锁定的 "POS-driven 模式 attitude 由 thrust-vector cascade 隐式推导" |
| E2 §4.5 | [E2 Controller 结构设计](E2-controller-structural.md) | 各环 `Controller_Cascade_Loop_Shell` 内部 `setpoint → error → law → output` 端口骨架;`PID_Block` / `AntiWindup_BackCalc` / `FeedForward_Composer` / `LowPass_Filter_1st` 实例位置 |

**Firmware 镜像引用:** 本文档不直接镜像 firmware 算法。`CONTROL_PARAM` 字段名引用记 `<pending hash>`,与 A3/A6/A7/B1/B2/B3 同期补齐(per 项目约定)。

## 4. 设计内容

### 4.1 控制律分类总表(每环一行)

| Loop ID | 工作名 | 控制律类 | 内部状态 | 前馈接入 | Anti-windup | 测量 / 误差滤波 | 输出(到下游环 / mixer)|
|---|---|---|---|---|---|---|---|
| **L-01** | Position(NED 外环) | **P-only** | 无 | `vel_ff`(`FMS_Out_Bus.vel_cmd_ned_mps`,gated by BIT_VEL co-set) | N/A(无 integrator) | 测量端无;P-only 无 D 项 | `vel_sp_inner[3]`(NED) → L-02 |
| **L-02** | Velocity(NED) | **PI + 加速度前馈** | `err_vel_int[3]` | `acc_ff`(`FMS_Out_Bus.acc_cmd_ned_mps2`,gated by BIT_ACC);**z 轴重力前馈 +g**(常驻);可选 thrust-vector geometry 推导(per E1 §4.3.5 Wave 8 锁定) | **back-calculation**(K_aw_vel) | velocity 测量 LPF(20 Hz,Tustin 离散 1 阶)| `acc_sp_inner[3]`(NED) → L-03 attitude derivation;同时 z 分量 → L-05 collective(thrust virtual) |
| **L-03** | Attitude(quaternion) | **P on quaternion error** | 无 | 通常无;yaw 子通道由 BIT_YAW 直接路由替换姿态环 yaw 分量(per E1 §4.3.3) | N/A(无 integrator) | 通常无(姿态测量 quat 已由 INS 平滑;200 Hz 下无 D 项)| `ang_rate_sp_inner[3]`(body) → L-04 |
| **L-04** | Angular rate(body, hot loop) | **PID(D-on-error,LPF mandatory)** | `err_rate_int[3]` + `err_rate_d_lpf_state[3]` + `err_rate_prev[3]` | 通常无;高级轨迹跟踪(yaw-to-thrust-vector)Phase 2 不启用 | **back-calculation**(K_aw_rate)+ saturation flag 由 L-05 mixer 上传(per E1 §4.3.4) | 角速度测量 LPF(30 Hz,Tustin)+ **D 项 LPF(30–50 Hz,mandatory)** | `moment_b_virtual[3]` + `thrust_virtual` → L-05 |
| **L-05** | Mixer / Allocator | **不是控制律 — 纯线性分配** | (mixer 内部状态由 E4 锁定) | N/A | N/A(saturation 上传给 L-04 / L-02 / L-01)| N/A | `motor_cmd[]` → Output Assembler |

**选型 trade-off 说明:**
- **L-01 P-only**:位置外环用 P 保持 cascade 简洁,不引入 integrator(避免与 L-02 integrator 共振 windup);稳态误差 / 风扰位移由 L-02 PI 的 `err_vel_int` 在速度层吸收(等价位置层有效"积分")。
- **L-02 PI + acc_ff**:速度环必须积分以消除常值风扰 / 倾角偏置 / 重力补偿误差;不引入 D 是因为 L-01 输出 `vel_sp_inner` 已经是经过 P 律的限幅信号,加 D 会放大 L-01 输出的离散阶跃噪声。
- **L-03 P-only**:姿态环用 P 让 cascade 阻尼依赖 L-04 rate 环的 D 项;P 律在 quaternion 误差上做 "small-angle 近似"足够稳健(细节见 §4.4)。
- **L-04 PID**:rate 环是稳定性根基,PID 给最快响应;P 提供刚性、I 消除舵面 / 推力偏置常值误差、D 提供阻尼防过冲;**D 项不可用未滤波信号**(角速度噪声直接通过 D 项放大成执行器抖动)。
- **L-05 不是律**:仅是 `B^{-1}` 几何线性映射 + 饱和裁剪;不需要 K_p/K_i/K_d。

### 4.2 L-01 Position loop 控制律

**输入:**
- 位置 setpoint:`pos_sp[3]` ← `FMS_Out_Bus.pos_cmd_ned_m`(经 §4.1.1 解码,gated by `BIT_POS` 与 `flag.position_valid`)
- 位置 measurement:`pos_meas[3]` ← `INS_Out_Bus.position_ned_m`
- 速度前馈(可选):`vel_ff[3]` ← `FMS_Out_Bus.vel_cmd_ned_mps`,**当且仅当** cmd_mask co-set `BIT_POS=1 AND BIT_VEL=1`(per D6 §4.3 表行 "POS+VEL 共置位 = POS 主 + VEL ff");否则 `vel_ff = 0`
- 增益:`K_p_pos_xy`(scalar,水平面共用)/ `K_p_pos_z`(scalar,竖直独立)— 来自 `CONTROL_PARAM`(B3 §4.6.4 字段名由 E4 leaf 兼容性核对)

**输出:** `vel_sp_inner[3]`(NED frame,m/s)馈入 L-02 `setpoint_inner` 端口(per E2 §4.5.2)。

**控制律(伪数学):**

```pseudo
// 误差(NED frame,m)
err_pos_xy = pos_sp[0:2] - pos_meas[0:2]
err_pos_z  = pos_sp[2]   - pos_meas[2]

// P 律 + 前馈
vel_sp_inner_xy = K_p_pos_xy * err_pos_xy + vel_ff[0:2]
vel_sp_inner_z  = K_p_pos_z  * err_pos_z  + vel_ff[2]

// 饱和(per-axis,limit 来自 CONTROL_PARAM)
vel_sp_inner = clamp(vel_sp_inner, -max_vel_*, +max_vel_*)
```

**Anti-windup:** N/A(P-only 无 integrator)。L-01 的"等效位置积分"由 L-02 `err_vel_int` 吸收。
**滤波器:** 测量端不引入 LPF — `INS_Out_Bus.position_ned_m` 已是 INS 滤波后估计;P-only 无 D 项,故无 D-LPF 需求。
**前馈策略:** 见 §4.8.1。

### 4.3 L-02 Velocity loop 控制律

**输入:**
- 速度 setpoint:`vel_sp[3]` ← (`use_external` 时)`FMS_Out_Bus.vel_cmd_ned_mps` 或 (`use_inner` 时)L-01 输出 `vel_sp_inner`(per E1 §4.3.2 / E2 §4.5.2)
- 速度 measurement:`vel_meas_raw[3]` ← `INS_Out_Bus.velocity_ned_mps`,经 §4.7.1 测量 LPF
- 加速度前馈(可选):`acc_ff[3]` ← `FMS_Out_Bus.acc_cmd_ned_mps2`,gated by `BIT_ACC=1`(per D6 §4.4 "ACC 必须 co-set with POS 或 VEL");否则 `acc_ff = 0`
- 重力前馈:`g_ff = [0, 0, +g]`(NED frame;z 向下为正,故 `+g` 抵消重力)— 常驻,与 cmd_mask 无关
- 增益:`K_p_vel_xy` / `K_p_vel_z` / `K_i_vel_xy` / `K_i_vel_z` / `K_aw_vel_xy` / `K_aw_vel_z` 来自 `CONTROL_PARAM`

**输出:** `acc_sp_inner[3]`(NED frame,m/s²)→ §4.10.1 thrust-vector cascade 派生 attitude reference;z 分量同时 → L-05 collective(per E2 §4.5.2 注)。

**控制律(伪数学,200 Hz 离散,Ts = 5 ms):**

```pseudo
// 测量滤波(per §4.7.1,Tustin 离散 1 阶 LPF,fc = 20 Hz)
vel_meas = LPF_20Hz(vel_meas_raw, state = vel_meas_lpf_state)

// 误差
err_vel = vel_sp - vel_meas

// 离散积分(forward Euler,与 anti-windup 联合更新见 §4.6.1)
err_vel_int_pre = err_vel_int + err_vel * Ts

// PI + 前馈 + 重力
acc_unsat_xy = K_p_vel_xy * err_vel[0:2] + K_i_vel_xy * err_vel_int_pre[0:2] + acc_ff[0:2]
acc_unsat_z  = K_p_vel_z  * err_vel[2]   + K_i_vel_z  * err_vel_int_pre[2]   + acc_ff[2] + g

acc_unsat = [acc_unsat_xy, acc_unsat_z]

// 饱和
acc_sat = clamp(acc_unsat, -max_acc_*, +max_acc_*)

// Anti-windup back-calculation(per §4.6.1)
sat_excess = acc_unsat - acc_sat
err_vel_int = err_vel_int_pre - K_aw_vel * sat_excess * Ts

// 输出
acc_sp_inner = acc_sat
```

**Anti-windup:** back-calculation,详见 §4.6.1;K_aw 按轴可独立(`K_aw_vel_xy` / `K_aw_vel_z`)。
**滤波器:** velocity 测量 LPF 1 阶 Tustin,fc = 20 Hz(详见 §4.7.1)。
**前馈策略:** acc_ff(条件)+ 常驻 g_ff(z 轴);见 §4.8.2。

### 4.4 L-03 Attitude loop 控制律(quaternion-based)

**输入:**
- 姿态 setpoint(主路径):`q_sp[4]` ← `FMS_Out_Bus.att_cmd_quat`(per B1 §4.7 + Wave 4 非阻塞建议:E3 锁 quat 为主路径;若 FMS 仅产 `att_cmd_euler_rad`,在 §4.1.1 解码侧用 Layer A `Quaternion_From_Euler` 转换为 `q_sp`)
- 姿态 measurement:`q_meas[4]` ← `INS_Out_Bus.quat_ned_to_b`
- yaw 子通道(可选直驱):`yaw_sp` ← `FMS_Out_Bus.yaw_cmd_rad`,gated by `BIT_YAW=1`(per E1 §4.3.3)
- 增益:`K_p_att_xy`(roll/pitch 共用,scalar)/ `K_p_att_z`(yaw 独立,scalar)— 来自 `CONTROL_PARAM`

**输出:** `ang_rate_sp_inner[3]`(body frame,rad/s)→ L-04 `setpoint_inner` 端口。

**控制律(quaternion 误差解算):**

```pseudo
// 1. quaternion 误差(NED→body desired vs actual)
//    q_err = q_sp ⊗ conj(q_meas)
q_err = Quaternion_Mul(q_sp, Quaternion_Conj(q_meas))   // Layer A 库块

// 2. 最短路径符号修正(避免绕远路 360°)
if q_err.w < 0:
    q_err = -q_err

// 3. quaternion → 角速度参考(等价于 small-angle 近似下的 axis-angle)
//    body-frame 角速度参考 = 2 * sgn(q_err.w) * vector_part(q_err) * K
//    sgn(q_err.w) 在步 2 修正后恒 = +1,简化为:
ang_rate_sp_xyz = 2 * [q_err.x; q_err.y; q_err.z]
ang_rate_sp_inner_xy = K_p_att_xy * ang_rate_sp_xyz[0:2]   // roll/pitch
ang_rate_sp_inner_z  = K_p_att_z  * ang_rate_sp_xyz[2]     // yaw

// 4. yaw 子通道直驱(BIT_YAW=1 时 override 上一步 yaw 分量)
if cmd_mask.BIT_YAW == 1:
    yaw_err = wrap_pi(yaw_sp - euler_yaw(q_meas))   // wrap to (-π, π]
    ang_rate_sp_inner_z = K_p_att_z * yaw_err

// 5. 饱和(per-axis,limit 来自 CONTROL_PARAM)
ang_rate_sp_inner = clamp(ang_rate_sp_inner, -max_rate_*, +max_rate_*)
```

**几何说明:** 步 3 中因子 `2` 来自 `q_err = [cos(θ/2), sin(θ/2)·n̂]` → small-angle 下 `vector_part ≈ θ·n̂/2`,故乘 2 还原 axis-angle。步 2 的最短路径修正等价于在 4-D 球面上始终选 `q_err.w ≥ 0` 的半球(避免 q 与 -q 等价时绕远 2π)。

**Anti-windup:** N/A(P-only 无 integrator)。
**滤波器:** 通常无 — `INS_Out_Bus.quat_ned_to_b` 已是 INS 平滑估计;200 Hz 下 q_err 噪声不足以驱动 D 项需求;P-only 无 D 项。
**前馈策略:** Phase 2 无 attitude FF。yaw-to-thrust-vector 高级 FF 留 Phase 3。
**与 thrust-vector cascade 的关系:** 当 cmd_mask 为 POS-driven(C-01/C-02)且 `BIT_ATT=0` 时,`q_sp` 不来自 FMS;由 L-02 输出 `acc_sp_inner` 经 §4.10.1 推导出,然后送入步 1。

### 4.5 L-04 Angular rate loop 控制律(hot loop)

**输入:**
- 角速度 setpoint:`ang_rate_sp[3]` ← (`use_external` 时)`FMS_Out_Bus.ang_rate_cmd_b_radps` 或 (`use_inner` 时)L-03 输出 `ang_rate_sp_inner`(per E1 §4.3.4)
- yaw 速率子通道(可选):`yaw_rate_sp` ← `FMS_Out_Bus.yaw_rate_cmd_radps`,gated by `BIT_YAW_RATE=1`(per E1 §4.3.4),override yaw 分量
- 角速度 measurement:`rate_meas_raw[3]` ← `INS_Out_Bus.ang_rate_b_radps`,经 §4.7.2 测量 LPF
- 增益:`K_p_rate_*` / `K_i_rate_*` / `K_d_rate_*` / `K_aw_rate_*`(每轴独立)— 来自 `CONTROL_PARAM`

**输出:** `moment_b_virtual[3]`(body frame,N·m 等效虚拟控制)+ `thrust_virtual`(若 L-02 不直接产出 thrust;Phase 2 多旋翼通常由 L-02 z 分量产出,L-04 仅出 moment)→ L-05 mixer。

**控制律(伪数学,200 Hz 离散):**

```pseudo
// 1. 测量滤波(per §4.7.2,Tustin 1 阶 LPF,fc = 30 Hz)
rate_meas = LPF_30Hz(rate_meas_raw, state = rate_meas_lpf_state)

// 2. 误差
err_rate = ang_rate_sp - rate_meas

// 3. 积分(与 anti-windup 联合更新)
err_rate_int_pre = err_rate_int + err_rate * Ts

// 4. D 项(D-on-error)+ 必须 LPF
//    raw D = (err_rate - err_rate_prev) / Ts
err_rate_d_raw = (err_rate - err_rate_prev) / Ts
err_rate_d = LPF_d(err_rate_d_raw, state = err_rate_d_lpf_state, fc = rate_d_lpf_cutoff_hz)
//    rate_d_lpf_cutoff_hz ∈ [30, 50] Hz(per E5 budget 协商;初值 30 Hz)
err_rate_prev = err_rate    // 状态更新置后步 8

// 5. PID 合成
moment_unsat = K_p_rate * err_rate
             + K_i_rate * err_rate_int_pre
             + K_d_rate * err_rate_d

// 6. 饱和(per-axis;limit 来自 CONTROL_PARAM 或 L-05 mixer 反馈)
moment_sat = clamp(moment_unsat, -max_moment_*, +max_moment_*)

// 7. Anti-windup back-calculation(per §4.6.2)
//    saturation 信号源:本环 moment 限幅 + L-05 motor_cmd 限幅(经
//    Controller_AntiWindup_Coordinator A8 §4.3.3 上传,per E1 §4.3.4 (c))
sat_excess = moment_unsat - moment_sat + sat_signal_from_mixer
err_rate_int = err_rate_int_pre - K_aw_rate * sat_excess * Ts

// 8. 状态更新与输出
moment_b_virtual = moment_sat
// err_rate_prev 已在步 4 更新
```

**Anti-windup:** back-calculation,详见 §4.6.2;saturation 信号双源(本环饱和 + mixer 上传)。
**滤波器:** 角速度测量 LPF 1 阶 Tustin,fc = 30 Hz(§4.7.2);**D 项 LPF 强制,1 阶 Tustin,fc = 30–50 Hz(§4.7.3)**。
**前馈策略:** 无(Phase 2)。
**架构 v1 §13 约束:** 单速率 5 ms,**禁用 variable-step**;所有滤波器必须显式离散化(Tustin),不得依赖 Simulink 自动连续→离散转换。

### 4.6 Anti-windup 算法

E3 在 L-02 与 L-04 两个含 integrator 环统一采用 **back-calculation** 方法。下方给出选型论证 + 公式。

**契约影响声明(2026-05-09 fix-up):** Back-calculation 需要 5 个新增 PARAM 字段:`K_aw_vel_xy`、`K_aw_vel_z`、`K_aw_rate_roll`、`K_aw_rate_pitch`、`K_aw_rate_yaw`,均为 R-T、`single`(float32)、默认值 0.5(vel)/0.3(rate)。这 5 个字段在 [B3 §4.5.2 CONTROL_PARAM](../B-contracts/B3-parameter-schema.md) 通过 RULES §10 sealed-amendment 同期登记(B3 字段总数 39 → 44),并在 INDEX 决策日志登记 contract impact。**B3 既有字段 `vel_int_lim_xy/z`(CONTROL_PARAM.10/11)与 `rate_int_lim_roll/pitch/yaw`(CONTROL_PARAM.26/27/28)保留作 hard-clamp safety 副本**(双层保护:back-calc 主路径 + clamp 兜底)。E3 §4.7.3 D-LPF 截止频率字段使用 B3 既有 `rate_d_lpf_cutoff_hz`(CONTROL_PARAM.29);Phase 2 三个 rate 轴共用同一 cutoff(若未来需要按轴独立 cutoff,触发新一轮契约修订)。

**候选方法对比:**

| 方法 | 行为 | 优点 | 缺点 |
|---|---|---|---|
| **Back-calculation**(选定)| 饱和时按 K_aw 比例把"未饱和-饱和"差值反扣到 integrator | 平滑、无脉冲;K_aw 给设计者一个"放电时间"调参旋钮(τ_aw ≈ K_p / (K_i · K_aw)) | 多一个 K_aw 调参 |
| Conditional integration(替代)| 饱和时直接冻结 integrator(不累加新误差) | 实现极简(一个 if) | 输出 jerky;退饱和瞬间可能触发"积分突变" |
| Clamping(`I_max`)| 对 integrator 状态本身做硬限幅 | 简单 | 不知道下游饱和点;在 K_p · err 已经能推到饱和时 integrator 仍可能继续累加 |

**E3 选定:back-calculation**(L-02 与 L-04 一致),理由:
1. 多旋翼控制器的执行器饱和(电机推力上限)频繁触发,Conditional integration 的 jerky 行为会被乘客 / 载荷感知。
2. K_aw 与 K_p / K_i 共同决定"反 windup 时间常数",可由 E4 leaf 调参经验值移植。
3. 与 PX4 / FMT firmware 历史实现一致(参考 PX4 mc_pos_control / mc_rate_control,K_aw 概念存在),便于 B4 contract diff 比较时不引入算法层差异。

#### 4.6.1 L-02 anti-windup(每轴独立)

```pseudo
// per §4.3 步骤(摘要)
err_vel_int_pre = err_vel_int + err_vel * Ts
acc_unsat = K_p_vel * err_vel + K_i_vel * err_vel_int_pre + acc_ff (+ g_ff_z)
acc_sat   = clamp(acc_unsat, ±max_acc)
sat_excess = acc_unsat - acc_sat
err_vel_int = err_vel_int_pre - K_aw_vel * sat_excess * Ts
```

**特性:**
- 当 `acc_unsat = acc_sat`(无饱和),`sat_excess = 0`,integrator 正常累加。
- 饱和深度越大 `sat_excess` 越大,反扣越强;K_aw_vel 越大,反 windup 越快(但过大可能产生过冲 → leaf 调参旋钮)。
- 每轴独立(`K_aw_vel_xy` / `K_aw_vel_z`),允许 z 轴(常受重力 / 推力饱和影响)更激进。

#### 4.6.2 L-04 anti-windup(每轴独立 + mixer 反馈)

L-04 的 saturation 信号有**两个来源**(per E1 §4.3.4 (c)):

1. **本环 moment 限幅**(`moment_unsat - moment_sat`):本环输出超出 `max_moment_*`。
2. **L-05 mixer motor 限幅上传**:`Controller_AntiWindup_Coordinator`(A8 §4.3.3)在 mixer 检测到 `motor_cmd[]` 触限时,把"哪个轴的 moment 实际不可达"反算成等效 `sat_signal_from_mixer[3]`,经 5 ms 同帧反馈给 L-04。具体反算几何由 [E4](E4-multicopter-leaf.md) 锁定(因依赖分配矩阵);E3 仅约定 saturation flag 接口与累加规则。

```pseudo
// per §4.5 步骤(摘要)
err_rate_int_pre = err_rate_int + err_rate * Ts
moment_unsat = K_p · err_rate + K_i · err_rate_int_pre + K_d · err_rate_d
moment_sat   = clamp(moment_unsat, ±max_moment)
sat_excess_local = moment_unsat - moment_sat
sat_excess_total = sat_excess_local + sat_signal_from_mixer
err_rate_int = err_rate_int_pre - K_aw_rate * sat_excess_total * Ts
```

**跨级 anti-windup 协调原则:**
- `Controller_AntiWindup_Coordinator` 一次只把 mixer saturation 反传给**最内的可吸收环**(典型 = L-04);若 L-04 已饱和且仍有未消化的 sat_signal,则继续上传给 L-02 的 `K_aw_vel` 路径(per E1 §4.3.5 (c))。
- E3 不在 L-01 / L-03 引入 K_aw_pos / K_aw_att(它们是 P-only,无 integrator;saturation 通过下游 P 反馈自然衰减)。

### 4.7 滤波器设计

所有滤波器**离散化方法统一为 Tustin(双线性)变换**,Ts = 5 ms(200 Hz 单速率,per 架构 v1 §13)。每个滤波器在 init / reset 后 state = 0(per A6 §4.2.3)。

**1 阶 Tustin LPF 通用形式:**

```pseudo
// 连续: H(s) = 1 / (τ·s + 1),  τ = 1 / (2π·fc)
// Tustin: s ← (2/Ts) · (z-1)/(z+1)
//
// 离散迭代:
//   α = (2 - Ts/τ) / (2 + Ts/τ)        // pole 项
//   β = (Ts/τ)    / (2 + Ts/τ)         // input 项
//   y[n] = α · y[n-1] + β · (u[n] + u[n-1])
//
// 注:Tustin 1 阶 LPF 引入"半步延迟"(group delay ≈ Ts/2 在低频),
//     比 forward Euler 稳定且无混叠折返。
```

#### 4.7.1 Velocity 测量 LPF(L-02)

- **截止频率:** `fc = 20 Hz`(默认值,可由 `CONTROL_PARAM` runtime override)。
- **位置:** L-02 measurement 端(`vel_meas_raw → vel_meas`)。
- **理由:** GPS / 视觉 / 多传感器融合后的 NED 速度典型噪声在 5–15 Hz 包络内;20 Hz 截止保留有用带宽,衰减高频噪声 ≥6 dB / oct。
- **状态:** 3-vector(每轴一个 1 阶滤波器)。

#### 4.7.2 角速度测量 LPF(L-04)

- **截止频率:** `fc = 30 Hz`(默认值)。
- **位置:** L-04 measurement 端(`rate_meas_raw → rate_meas`)。
- **理由:** Plant 发布原始 `ang_rate_b_radps`(架构指引:Plant 不预滤波);Controller 侧统一截止 30 Hz 既抑制 IMU 高频噪声,又不破坏 200 Hz 控制带宽(`fc < f_nyquist/2 = 50 Hz`)。
- **状态:** 3-vector。

#### 4.7.3 D 项 LPF(L-04,**强制**)

- **截止频率:** `fc ∈ [30, 50] Hz`,默认 30 Hz;具体值由 [E5](E5-performance-budget.md) 在性能预算评估后微调,落到 `CONTROL_PARAM.rate_d_lpf_cutoff_hz`。
- **位置:** L-04 D 项支路(`err_rate_d_raw → err_rate_d`)。
- **强制理由:** 未滤波的 D 项是噪声放大器:`D ≈ K_d · n[t]/Ts` 中 `n[t]` 哪怕是 `INS_Out_Bus` 量化噪声(LSB ~1e-3 rad/s)经 `K_d/Ts = K_d · 200 Hz` 放大都会在 motor_cmd 上产生可听见的抖动。
- **离散化:** 同 §4.7 通用形式;**注意:** 离散 D 项 LPF 的等价形式为 `(LPF_d) ∘ (forward-difference)`,可直接用一个状态量 `err_rate_d_lpf_state` 实现(等价于"带极点的差分"):
  ```pseudo
  α_d = (2 - Ts·ω_d) / (2 + Ts·ω_d),  ω_d = 2π·fc
  y_d[n] = α_d · y_d[n-1] + (1-α_d)/Ts · (err_rate[n] - err_rate[n-1])
  ```

**滤波器统一不变量:**
- 所有滤波器 init/reset state = 0(per A6 §4.2.3 Controller 项 1)。
- 算法层不允许"跨步保留 state"作为隐式存储;状态由 E2 §4.5 `LowPass_Filter_1st` 实例显式持有。
- 不引入 IIR 高阶 / Butterworth — Phase 2 1 阶 Tustin 足够。
- 一步采样延迟在 5 ms 周期内可接受(group delay ≈ 2.5 ms);phase margin 影响留 [E5](E5-performance-budget.md) 复核。

### 4.8 前馈策略

#### 4.8.1 L-01 速度前馈

- **来源:** `FMS_Out_Bus.vel_cmd_ned_mps`
- **激活条件:** cmd_mask `BIT_POS=1 AND BIT_VEL=1`(co-set;per D6 §4.3 表 — POS+VEL 时 POS 主、VEL 作前馈)
- **接入点:** `vel_sp_inner = K_p_pos · err_pos + vel_ff`(per §4.2)
- **失活回退:** 若 `BIT_VEL=0` 或 `flag.velocity_valid=0`,`vel_ff = 0`(zero-frozen,per E1 §4.4.2)
- **意义:** 在轨迹跟踪场景(D4 jerk-limited shaper 输出 pos+vel+acc 三元组)消除 P 律的"跟随时间常数"延迟。

#### 4.8.2 L-02 加速度前馈(条件)+ 重力前馈(常驻)

**加速度前馈(条件):**
- **来源:** `FMS_Out_Bus.acc_cmd_ned_mps2`
- **激活条件:** cmd_mask `BIT_ACC=1`,且必须与 `BIT_POS` 或 `BIT_VEL` co-set(per D6 §4.4)
- **接入点:** `acc_unsat = K_p_vel · err_vel + K_i_vel · err_vel_int + acc_ff + g_ff`
- **失活回退:** `BIT_ACC=0` → `acc_ff = 0`

**重力前馈(常驻):**
- **来源:** 常量 `g = 9.80665 m/s²`(NED z 向下为正,故 `+g` 抵消重力)
- **激活条件:** **始终激活**,与 cmd_mask 无关(只要 L-02 active)
- **接入点:** z 分量:`acc_unsat_z = K_p_vel_z · err_vel_z + K_i_vel_z · err_vel_int_z + acc_ff_z + g`
- **意义:** 让 L-02 输出的"等效推力加速度"已包含悬停推力分量,L-02 integrator 仅累加扰动(质量误差 / 风扰),减小 windup 概率。
- **算法侧约束:** g 是常量(不是 `INS_Out_Bus.grav_*` 字段;Phase 2 不消费 INS 重力估计)。

#### 4.8.3 L-03 / L-04 前馈

Phase 2 **无前馈**:
- L-03 attitude FF(从 D4 feed-forward attitude 派生)— 留 Phase 3。
- L-04 yaw-to-thrust-vector / 角加速度 FF — 留 Phase 3。

#### 4.8.4 thrust-vector cascade 推导(L-02 → L-03 attitude reference)

per [E1 §4.3.5](E1-controller-functional.md) Wave 8 锁定:**POS-driven 模式下 attitude 由 `acc_sp_inner` 隐式推导**,不来自 FMS `att_cmd_quat`(即 `BIT_ATT=0`)。算法上:

```pseudo
// L-02 输出(NED frame,m/s²):
acc_sp_inner = [a_n, a_e, a_d]    // a_d 已含 +g

// 期望"thrust 方向"(body z 轴在 NED 下的取向):
thrust_dir_ned = -acc_sp_inner / norm(acc_sp_inner)   // 单位向量,指向 -a_d

// 与 yaw_sp(从 FMS_Out_Bus.yaw_cmd_rad 或上一帧保持值)合成:
q_sp = Quaternion_From_ThrustDir_And_Yaw(thrust_dir_ned, yaw_sp)   // Layer A 库块

// 馈入 L-03 步 1(替换 FMS 提供的 q_sp 路径)
```

E3 仅声明算法存在 + 输入输出端口语义;`Quaternion_From_ThrustDir_And_Yaw` 的具体几何(SO(3) 主轴对齐 + yaw 重排)由 [E4](E4-multicopter-leaf.md) 锁定(多旋翼几何相关)。

### 4.9 Init / reset 算法侧稳态(per A6)

对照 [A6 §4.2.3](../A-architecture/A6-init-reset-contract.md) Controller 项 1 + §4.4.2 reset 后稳态契约,E3 在算法侧承诺:

| 状态量 | 所属环 | Controller_init 后 | reset 后 |
|---|---|---|---|
| `err_vel_int[3]` | L-02 | 0 | 0(等价 init) |
| `err_rate_int[3]` | L-04 | 0 | 0 |
| `err_rate_prev[3]` | L-04 D 项 | 0 | 0 |
| `vel_meas_lpf_state[3]` | L-02 测量 LPF | 0 | 0 |
| `rate_meas_lpf_state[3]` | L-04 测量 LPF | 0 | 0 |
| `err_rate_d_lpf_state[3]` | L-04 D 项 LPF | 0 | 0 |
| `q_err_prev`(若引入)| L-03 | identity quaternion `[1,0,0,0]` | identity |

**Bypass-state 策略(per E1 §4.4.2):** zero-frozen + integrator clear。即:当某环因 cmd_mask / valid-gate 处于 bypass 状态时,**该环的 integrator 必须被清零**(不仅是输出置 0),避免再激活时 windup 残留产生瞬态。E3 在每环 `reset` 端口与"valid_gate=false 持续超过 1 帧"两条触发路径上都拉高 integrator clear。

**LPF state 在 bypass 时的处理:** **保留**(不清零) — 避免 mode 切换 / re-arm 后 LPF 重新启动产生瞬态(典型现象:刚 unfrozen 的 LPF 输出会"弹"到 measurement 当前值)。这与 integrator 的"必须清零"策略相反,理由:
- Integrator 的 windup 是**累积型危害**(残留误差越积越大);清零是安全的。
- LPF state 是**过去测量的平滑残留**,清零反而引入瞬态;保留可让 mode 切换时 measurement 路径连续。

### 4.10 与下游的交叉引用

| 下游 | 关系类型 | E3 提供给下游的输入 |
|---|---|---|
| [E4 多旋翼 leaf](E4-multicopter-leaf.md) | E3 → E4 | 各环控制律方程 + 增益槽位列表(`K_p_pos_*` / `K_p_vel_*` / `K_i_vel_*` / `K_aw_vel_*` / `K_p_att_*` / `K_p_rate_*` / `K_i_rate_*` / `K_d_rate_*` / `K_aw_rate_*` / `rate_d_lpf_cutoff_hz`)+ saturation 信号几何接口约定;E4 在此基础上填入数值 + 分配矩阵反算几何 |
| [E5 性能预算](E5-performance-budget.md) | E3 → E5 | 各环算法的运算规模(L-04 PID + 3 个 LPF + anti-windup ≈ 最热)+ 滤波器实例数 + Tustin 实现成本估算;E5 据此判断 5 ms 周期内执行可行性 + 决定是否需要 LUT / 定点 |
| [B3 CONTROL_PARAM](../B-contracts/B3-parameter-schema.md) | E3 ⇢ B3 | E3 锁定增益**结构**(每轴独立 / xy 共用 / z 独立的划分),B3 已分配 39 字段槽位;E4 leaf 兼容性核对时确认字段名 |

## 5. 已知风险与悬而未决问题

- **D 项 LPF 截止频率 30 vs 50 Hz**
  - 影响:E5 性能预算 + E4 实测调参
  - 处置:E3 锁定 default = 30 Hz + `CONTROL_PARAM.rate_d_lpf_cutoff_hz` runtime override;E5 复核后若 phase margin 不足允许 leaf 上调到 50 Hz。

- **L-02 thrust 路径 vs attitude 路径的"虚拟控制"分配**
  - 影响:L-02 输出是 `acc_sp_inner[3]` 还是分裂为 `thrust_scalar + tilt_acc[2]`?
  - 当前选型:统一 `acc_sp_inner[3]`;§4.8.4 推导 thrust_dir + scalar magnitude 分两路注入 L-03 / L-05。
  - 处置:Phase 2 锁定如本文 §4.3 + §4.8.4;若 E4 在多旋翼 leaf 中发现"thrust scalar 单独传更高效",E5 复核时可重订接口(算法不变,仅信号包装)。

- **mixer saturation 反算几何依赖分配矩阵**
  - 影响:§4.6.2 的 `sat_signal_from_mixer[3]` 反算具体公式
  - 处置:E3 仅约定接口与累加规则;具体几何 = E4 锁定;E3 / E4 在 Wave 9 联调。

- **`Quaternion_From_ThrustDir_And_Yaw` 库块归属**
  - 影响:Layer A vs Layer B(per A8)
  - 处置:作为多旋翼专用的几何块(SO(3) 主轴对齐 + yaw 重排只在 thrust-vector cascade 出现,非通用旋转工具)— 倾向 Layer B(`model/controller/shared/`),具体由 A8 在 Wave 9 复核时决定;E3 仅声明算法。

- **att_cmd_euler vs att_cmd_quat 二选一**(B1 Wave 4 非阻塞建议)
  - 影响:解码侧路径数(1 或 2)
  - 处置:E3 锁 quat 为主;若 FMS 仅产 euler,在 §4.1.1 解码侧用 `Quaternion_From_Euler` 转换,L-03 内部仅消费 quat;不在 L-03 内部引入两条路径。

- **g_ff 是用常量还是 INS 估计?**
  - 影响:不同纬度 / 海拔的悬停推力偏差(地球 g 变化 ~0.5%)
  - 处置:Phase 2 用常量 9.80665(算法简单,误差 < 0.5% 由 L-02 integrator 吸收);Phase 3 若有 INS_Out_Bus 重力字段可改;E3 不引入 INS 重力字段消费(避免在 §4.7 中又引入一个 valid gate)。

## 6. 退出条件复核

对照 [`00-design-plan.md`](../00-design-plan.md) E3 行"退出条件":**位置/速度/姿态/角速度环数学定义、前馈、抗 windup、滤波器**。

| # | 退出条件原文 | 本文档依据 | 状态 |
|---|---|---|---|
| 1 | 位置环数学定义 | §4.2(L-01 P-only,误差 / P 律 / 饱和 / 状态) | 满足 |
| 2 | 速度环数学定义 | §4.3(L-02 PI,离散积分 / 输出装配 / 状态) | 满足 |
| 3 | 姿态环数学定义 | §4.4(L-03 P on quaternion,quat 误差解算 + 最短路径 + yaw 子通道) | 满足 |
| 4 | 角速度环数学定义 | §4.5(L-04 PID hot loop,测量 LPF + D 项 LPF + saturation 双源) | 满足 |
| 5 | 前馈 | §4.8(L-01 vel_ff / L-02 acc_ff + g_ff / L-04 无 / thrust-vector cascade 派生 attitude) | 满足 |
| 6 | 抗 windup | §4.6(选型 trade-off + L-02 / L-04 back-calculation 公式 + 跨级 coordinator 协议) | 满足 |
| 7 | 滤波器 | §4.7(velocity LPF 20 Hz / rate LPF 30 Hz / D 项 LPF 30–50 Hz mandatory + Tustin 离散化通用形式) | 满足 |

## 7. 下游影响

按 [`01-design-relationships.md`](../01-design-relationships.md) E3 出边(`E3 → E4`、`E3 → E5`):

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [E4 多旋翼 leaf](E4-multicopter-leaf.md) | E3 → E4(强前置) | §4.1 控制律分类 + §4.2–§4.5 各环方程 + §4.6 anti-windup 算法 + §4.7 滤波器 + §4.8 前馈接入点 + §4.10 增益槽位列表;E4 在此基础上填入多旋翼 `CONTROL_PARAM` 数值 + 分配矩阵反算 saturation 几何 + thrust-vector cascade 几何块 |
| [E5 性能预算](E5-performance-budget.md) | E3 → E5(强前置) | §4.5 hot loop 算法规模(PID + 3 LPF + back-calc)+ §4.7 滤波器实例数 + Tustin 离散化实现成本 + §4.4 quaternion 误差解算开销;E5 据此评估 5 ms 周期可行性 + 决定 LUT / 定点 / MATLAB Function 禁用范围 |

非直接下游但受影响:
- [E2](E2-controller-structural.md):E3 的算法选型反过来确认 E2 §4.5 各环 `Controller_Cascade_Loop_Shell` 槽位编排兼容(error → law → output 框架被全部环路接受;LPF 实例位置在 measurement 端 + D 项支路);若 E3 算法触发 E2 骨架补充,在 E2 变更日志追加 — 本次未触发。
- [B3 CONTROL_PARAM](../B-contracts/B3-parameter-schema.md):E3 锁的"每轴独立 / xy 共用 / z 独立"增益结构与 B3 §4.6.4 字段集兼容;E4 leaf 兼容性核对时引用本文 §4.10。

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-09 | E3 author | 初稿 — 5 环控制律分类 + L-01..L-04 数学方程 + back-calculation anti-windup + Tustin 滤波器 + 前馈策略 + thrust-vector cascade 派生 + init/reset 算法侧稳态 |
| 2026-05-09 | fix-up author | Wave 9 reviewer I-1 修复:contract_impact `no → yes`(back-calculation 需要 5 个新增 PARAM 字段 K_aw_vel_xy/z + K_aw_rate_roll/pitch/yaw,通过 B3 §4.5.2 sealed-amendment 同期登记);§4.6 添加契约影响声明段;`alpha_d_cutoff_hz` 重命名为 `rate_d_lpf_cutoff_hz`(对齐 B3 既有 CONTROL_PARAM.29);保留 `*_int_lim_*` 作 hard-clamp safety 副本。Self-check 6/7 由 N/A → 显式登记。 |

## Self-check

- [x] frontmatter 完整,字段值合法(work_item=E3 / contract_impact=yes / status=draft / upstream 全部 reviewed)
- [x] 上游文档全部存在且 status ≥ reviewed:架构 v1 / A6 / A7 / B1 / B2 / B3 / D6 / E1 / E2 全部 reviewed(per INDEX 已完成清单)
- [x] 退出条件逐条复核完成,每条均给出依据(§6 表 7 行全 "满足")
- [x] 引用路径全部可点击访问(架构 v1 / A6 / A7 / B1 / B2 / B3 / D6 / E1 / E2 / E4 / E5 / 00-design-plan / 01-design-relationships)
- [x] 不存在 RULES §5 禁则中的内容(无 .slx / 无 .m 可执行 / 伪代码全部标 `pseudo` / firmware commit hash 占位 `<pending hash>` 与项目约定一致;不重复 bus/enum/param schema)
- [x] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记 — 2026-05-09 Wave 9 fix-up:back-calculation anti-windup 需 5 个新增 K_aw_* 字段 → 通过 B3 §4.5.2 sealed-amendment 登记 + INDEX 决策日志(orchestrator 在 Wave 9 finalize 时同期登记)
- [x] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径 — 本文不直接镜像 firmware;`CONTROL_PARAM` 字段引用通过 B3 间接,B3 §3 已记录 `FMT-Firmware/src/model/control/<vehicle>/lib/Controller_types.h` + commit hash 占位 `<pending hash>` 与 A3/A6/A7/B1/B2 同 batch 模式
- [x] 下游影响已沿关系图识别完毕(§7:E3 → E4、E3 → E5;E2 受影响但未触发 §4.5 骨架补充)
- [x] 文档不超出本工作项范围(无越权设计):未定义级联拓扑(E2)/ 未定义 cmd_mask 裁剪规则(E1)/ 未定义多旋翼分配矩阵(E4)/ 未定义性能预算(E5)/ 未重复定义 bus/enum/param schema(B 区)
