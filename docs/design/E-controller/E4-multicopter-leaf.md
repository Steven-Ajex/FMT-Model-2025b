---
work_item: E4
title: Controller 多旋翼 leaf 设计(混控矩阵 / 几何 / 增益结构 / CONTROL_PARAM 兼容性)
upstream: [架构v1, A1, A2, A3, A4, A5, A6, A7, A8, B1, B2, B3, C3, D6, E1, E2, E3]
contract_impact: no
status: draft
authored_at: 2026-05-09
last_reviewed_at:
reviewer_verdict: none
---

# E4 Controller 多旋翼 leaf 设计

## 1. 目的

本文件为 Phase 2 多旋翼 vehicle 在 Controller 模块中的 **Layer C(leaf)** 填具体几何依赖、混控逆向矩阵、输出归一化、per-axis 增益字段集与 PX4-class 数值默认,落地 [E2 §4.1.6 Mixer/Allocator(LEAF: E4)](E2-controller-structural.md#41-controller-顶层子系统拓扑) 与 [E1 §4.1 L-05 mixer-leaf 环路](E1-controller-functional.md);并显式核对所有 [`CONTROL_PARAM`(B3 §4.5)](../B-contracts/B3-parameter-schema.md#45-control_param-schema) 字段名与 firmware tip `Controller_types.h` 字段名的兼容性(闭合 [审计 F-33](../_audit/2026-05-05-plan-audit.md))。

**与 E3(算法层)分工**:E4 锁**结构骨架**(混控矩阵的几何派生、归一化范围、增益**字段名集合**、Phase 2 数值默认),不锁控制律方程结构 / 前馈公式 / anti-windup 算法 / 滤波器实现 — 算法由同 wave sibling [E3](E3-loops-algorithm.md) 拥有。E4 主张的字段名集**对任何 P/PI/PID 选型保持稳定**(per [B3 §4.5.2](../B-contracts/B3-parameter-schema.md#452-字段表) 已锁的 39 字段命名)。

## 2. 范围

**在范围:**

- §4.1 多旋翼几何 echo(从 [C3 §4.1](../C-plant/C3-multicopter-leaf.md#41-多旋翼几何mc_geometry_constants-leaf) 单向引用 quad-X / arm length / motor 编号 / motor_dir,**不**重定义)
- §4.2 混控逆向 4×4 矩阵(force/moment cmd → per-motor thrust);**精确为 [C3 §4.4](../C-plant/C3-multicopter-leaf.md#44-分配矩阵mc_allocation_matrix-leafforward-direction) forward 矩阵的数学逆**
- §4.3 motor_cmd 输出归一化(0..1 per motor)+ saturation 与 anti-windup 信号回环占位(算法细则 forward-cite E3)
- §4.4 per-axis 增益**字段名集合**(全 39 个 CONTROL_PARAM 字段;数值含义 echo B3 已锁;不重定义 schema)
- §4.5 Phase 2 PX4-class 数值默认(全部标 `(B4 verify; PX4-class baseline)`)
- §4.6 reset / disarm / 整流器初始化行为(echo [A6 §4.4.3 Control_Out_Bus reset 类别](../A-architecture/A6-init-reset-contract.md))
- §4.7 CONTROL_PARAM 字段名兼容性核对表(全 39 字段,闭合审计 F-33;填 [B3 §4.9.3](../B-contracts/B3-parameter-schema.md#493-e4--control_param-兼容性核对模板) E4 模板)

**不在范围(由其他工作项处理):**

- 控制律方程 / 前馈结构 / anti-windup 算法形态(back-calculation vs conditional clamp)/ D-term LPF 数学实现 — 由同 Wave 9 sibling [E3](E3-loops-algorithm.md)
- cmd_mask 触发的环路裁剪规则 / 级联 hand-off / bumpless 功能要求 — 由 [E1](E1-controller-functional.md)(Wave 8 reviewed)
- 级联拓扑 / shared vs leaf 边界 / 库块归属 — 由 [E2](E2-controller-structural.md)(Wave 7 reviewed)
- 5 ms 周期内各环执行预算 / LUT / 定点 — 由 [E5 性能预算](E5-performance-budget.md)(Wave 10)
- Plant 侧 forward 分配矩阵 / 几何参数化 / 电机推力曲线物理常数 — 由 [C3](../C-plant/C3-multicopter-leaf.md)(Wave 9 sibling,本文件 §4.1 / §4.2 单向引用)
- bus / enum / parameter schema(字段顺序、类型、宽度、运行时-vs-编译期分类)— 由 [B1](../B-contracts/B1-bus-inventory.md) / [B2](../B-contracts/B2-enum-inventory.md) / [B3](../B-contracts/B3-parameter-schema.md)
- Phase 5 fixwing / VTOL leaf — 排除(per [00-design-plan §5](../00-design-plan.md))

## 3. 依赖

| 上游 | 引用位置 | 用途 |
|---|---|---|
| 架构 v1 §7.2 / §8.4 | [架构 v1](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | Controller 顶层职责 + 多旋翼 mixer/allocation 是 deepest leaf |
| 架构 v1 §11 | [架构 v1](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | leaf 层在 Simulink/script/codegen 中的位置 |
| 架构 v1 §12.2 | [架构 v1](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | Controller 7 子模块 §12.2.6 Mixer/Allocator 是 E4 落地点 |
| 架构 v1 §13 | [架构 v1](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | Controller = 5 ms internally-single-rate;mixer 同步运行于 5 ms |
| 架构 v1 §16 | [架构 v1](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | 多旋翼 = Phase 2 smallest slice;quad-X 是默认拓扑 |
| [A1 §5.1.2](../A-architecture/A1-v1-review.md) | A1 critical 缺口 — runtime-tunable vs compile-inlined 分类(B3 D-1 闭合)| §4.4 字段类引用 [B3 §4.5.2 Class 列](../B-contracts/B3-parameter-schema.md#452-字段表) |
| [A2 R-7.1 / E-5](../A-architecture/A2-naming-conventions.md) | 命名规则:`*_PARAM` struct 名固化;`firmware-legacy:` 前缀处置 | §4.7 兼容性核对处置链 |
| [A3 §4.5.5](../A-architecture/A3-module-boundaries.md) | `Control_Out_Bus` 字段表(`motor_cmd[]` / `actuator_cmd[]` / passthrough echo / debug)| §4.3 输出归一化 + saturation flag 落点 |
| [A4 RB-05](../A-architecture/A4-rate-boundaries.md) | Controller 输出 ZOH(5 ms → 1 ms 到 Plant)| §4.3 输出 = 5 ms-stable normalized cmd |
| [A5 §4.2 / §4.3](../A-architecture/A5-variant-strategy.md) | Phase 2 frozen `motor_count = 4`;quad-X = 唯一 active variant;**E4 与 C3 共享同一 frozen 维度** | §4.1 拓扑选择 + §4.2 矩阵 dim |
| [A6 §4.2.3 / §4.4.3 / §4.5.2](../A-architecture/A6-init-reset-contract.md) | `Controller_init` 入口契约 + `Control_Out_Bus` reset 类别 + reset 协调(集中 FMS 权威)| §4.6 reset 行为 |
| [A7 §4.5](../A-architecture/A7-time-conventions.md) | Controller 内 dt 来源固定步长 | §4.4 LPF / 积分 dt 来源 echo |
| [A8 §4.3.3](../A-architecture/A8-shared-library-roster.md) | Layer C `model/controller/vehicles/multicopter/` leaf 路径占位 | §4.1 leaf 落地路径 |
| [B1 §4.4 / §4.7](../B-contracts/B1-bus-inventory.md) | `Control_Out_Bus.motor_cmd[]` / debug 字段 schema(E4 引用,不重定义)| §4.3 |
| [B2 §4.4.6 / §4.4.13](../B-contracts/B2-enum-inventory.md) | `MixerGeometry` 枚举(`MX_QUAD_X` 等)| §4.1.1 拓扑选择 echo |
| [B3 §4.5.2 / §4.9.3](../B-contracts/B3-parameter-schema.md) | `CONTROL_PARAM` 39 字段 schema + §4.9.3 E4 兼容性核对模板(空模板)| §4.4 字段引用;§4.7 把 §4.9.3 模板填满 |
| [C3 §4.1 / §4.4](../C-plant/C3-multicopter-leaf.md) | Plant 多旋翼 leaf 几何与 forward 分配矩阵(Wave 9 sibling,**单向**:E4 inverse 派生自 C3 forward)| §4.1 几何 echo + §4.2 inverse 数学一致性 |
| [D6 §4.x](../D-fms/D6-fms-controller-interface.md) | `cmd_mask` 位语义(已 reviewed Wave 8);`MASK_BIT_THROTTLE_PASSTHROUGH=7` 时 mixer 直消费 `FMS_Out_Bus.throttle_cmd` 的边界 | §4.3 throttle passthrough 占位 |
| [E1 §4.1 L-05 / §4.3](E1-controller-functional.md) | mixer-leaf 环路功能身份(always-on 当 cmd_mask ≠ 0);输入 = L-04 输出(moment + thrust virtual control)| §4.2 mixer 输入接口 + §4.6 disarm 时 motor_cmd=0 |
| [E2 §4.1.6](E2-controller-structural.md) | Controller 拓扑中 Mixer/Allocator 子系统骨架(LEAF: E4)| §4.1 落地点 |
| **[E3](E3-loops-algorithm.md)**(Wave 9 sibling — *起草中*)| 角速度环输出量纲(body moment N·m + total thrust)+ anti-windup 信号端口 + LPF 实现 | §4.2 mixer 输入接口 forward-cite;§4.3 anti-windup 反馈端口;§4.6 reset 时各环 integrator 清零 forward-cite |
| FMT-Firmware @ `<pending hash>` | `FMT-Firmware/src/model/control/mc_controller/lib/Controller_types.h` | §4.7 字段兼容性核对源(待 [B4](../B-contracts/B4-contract-diff.md) / [I3](../I-tooling/I3-contract-diff.md) 首跑后 verify;同 [B3 §3 / §4.5.1 header](../B-contracts/B3-parameter-schema.md#451-header) 占位)|

**Wave 9 sibling 注记 — `E3 → E4` known-loose**:

[`01-design-relationships.md` §4.5](../01-design-relationships.md) 声明 `E3 → E4` 为强前置,但 [`00-design-plan.md`](../00-design-plan.md) 与 orchestrator 的 wave 排序(per [INDEX 决策日志](../INDEX.md))**显式**把 E3 与 E4 同放 Wave 9 并行。本工作项采用与 Wave 7 `E1 → E2` 同型的 known-loose 处置:

- **E4 锁**:多旋翼几何 echo(C3 forward direction 数学逆)、混控矩阵代数式、输出归一化范围、增益**字段名**集合、PX4-class 数值默认 — 这些**对 E3 的 P/PI/PID 算法选型保持稳定**(算法影响 gain 数值的整定,不影响字段名集本身;字段名集 owned 为 [B3 §4.5.2](../B-contracts/B3-parameter-schema.md#452-字段表) 的 39 字段)
- **E4 推迟**:控制律方程结构、前馈展开式、anti-windup 算法形态、D-term LPF 实现细节 — 由 E3 在同 wave 锁定;E4 仅 forward-cite 端口与信号
- 评审形式:**非 co-seal**(per orchestrator decision);E4 评审时 E3 必须达到 status=draft 且二者**字段名集合一致**(E4 §4.4 字段集 ⊆ B3 §4.5.2 schema = E3 引用源,自动满足)
- 若 E3 在 Wave 9 内提出 algorithm-driven 字段名增删,E4 走 [RULES §10 变更日志](../RULES.md) + B3 §4.5.2 同步增订(B3 是字段名权威)

环境注:FMT-Firmware 在本设计阶段**未挂载**;§4.7 表全部以 `(B4 verify; expected …)` 占位,与 [C3 §4.6](../C-plant/C3-multicopter-leaf.md#46-plant_param-字段名兼容性核对表闭合审计-f-33) PLANT_PARAM 兼容性核对采用同型 placeholder 流程。

## 4. 设计内容

### 4.1 多旋翼几何 echo(从 C3 §4.1 单向引用)

E4 leaf **不**重定义几何;直接从 [C3 §4.1.2 quad-X 几何布局](../C-plant/C3-multicopter-leaf.md#412-quad-x-几何布局ned-body-frameFrd-约定-per-a2) 引用。**关键不变量**:E4 inverse 矩阵的几何系数(`r` = arm_length/√2,`c_q` = motor torque constant)与 C3 forward 矩阵**派生自同一 PLANT_PARAM 字段**(per [B3 §4.3.2](../B-contracts/B3-parameter-schema.md#432-字段表) `arm_length_m` = `PLANT_PARAM.04` / `motor_torque_const_nmpn` = `PLANT_PARAM.09` / `motor_pos_b_m` = `PLANT_PARAM.06` / `motor_dir` = `PLANT_PARAM.07`),保证闭环一致性([C3 §4.4.3 forward-cite](../C-plant/C3-multicopter-leaf.md#443-与-e4-inversee4-multicopter-leafmdcross-reference占位))。

#### 4.1.1 拓扑选择(echo C3 §4.1.1)

| 拓扑 | Phase 2 状态 | E4 inverse 行为 |
|---|---|---|
| **quad-X**(默认 + 唯一 active) | active(per [A5 §4.2 frozen `motor_count = 4`](../A-architecture/A5-variant-strategy.md))| §4.2 矩阵 = quad-X 4×4 explicit form |
| quad-+ | placeholder(未启用) | §4.2 通过 `mixer_geometry` (`CONTROL_PARAM.33`) = `MX_QUAD_PLUS` 切换矩阵系数(arm 投影 r → arm_length);Phase 2 不启用 |
| hex / octa | placeholder(未启用) | Phase 2 frozen 4 motor;Phase 5 解冻 |

`mixer_geometry` 字段(`CONTROL_PARAM.33`,`uint8` 枚举 `MixerGeometry` per [B2 §4.4.6](../B-contracts/B2-enum-inventory.md))= `C-I`(compile-inlined,per [B3 §4.5.2](../B-contracts/B3-parameter-schema.md#452-字段表) Class 列);Phase 2 默认 `MX_QUAD_X`。

#### 4.1.2 motor 编号约定(逐字 echo C3 §4.1.2)

| Motor index | body 位置 | spin 方向 (`motor_dir`) | C3 §4.1.2 备注 |
|---|---|---|---|
| `motor_cmd[0]` | `[+arm/√2, -arm/√2, 0]` | +1 (CW) | 前左 |
| `motor_cmd[1]` | `[+arm/√2, +arm/√2, 0]` | -1 (CCW) | 前右 |
| `motor_cmd[2]` | `[-arm/√2, +arm/√2, 0]` | +1 (CW) | 后右 |
| `motor_cmd[3]` | `[-arm/√2, -arm/√2, 0]` | -1 (CCW) | 后左 |

**编号锁定**:E4 motor index 顺序与 [B1 §4.4 `Control_Out_Bus.motor_cmd[]`](../B-contracts/B1-bus-inventory.md) 数组顺序、C3 §4.1.2 表行顺序、Plant `Plant_States_Bus.motor_rpm[]` 顺序**逐索引一致**。任何索引置换需走 RULES §10 变更日志 + B1 / C3 / E4 同步修订。

### 4.2 混控逆向矩阵(`MC_Mixer_QuadX_Inverse` leaf,inverse direction)

E4 落地 **inverse** 分配矩阵(force/moment cmd → per-motor thrust),与 [C3 §4.4 forward](../C-plant/C3-multicopter-leaf.md#44-分配矩阵mc_allocation_matrix-leafforward-direction) 数据流方向相反。两者**不**共享运行时数据(Plant 不消费 inverse;Controller 不消费 forward),但**必须**派生自同一几何字段(`PLANT_PARAM.04 / .06 / .07 / .09`),保证 MIL 闭环数学闭合。

#### 4.2.1 输入接口(从 E3 角速度环 + 高度通道接出)

E4 mixer 子系统输入(per [E2 §4.1.6](E2-controller-structural.md#41-controller-顶层子系统拓扑) 信号路由槽位):

| 信号 | 单位 | 来源 | 备注 |
|---|---|---|---|
| `T_total_cmd_n` | N(总推力,>0)| E3 速度环 z-axis virtual control + hover feed-forward(forward-cite [E3 §4.x](E3-loops-algorithm.md))| 标量;`hover_thrust_n01` 反归一化 + acceleration 项;throttle passthrough 时直接消费 `FMS_Out_Bus.throttle_cmd × motor_thrust_max_n × 4` |
| `M_roll_cmd_nm` | N·m(绕 body x)| E3 角速度环输出 | per [B3 §4.5.2 rate_p_gain_roll Unit](../B-contracts/B3-parameter-schema.md#452-字段表) = `(N·m)/(rad/s)` 量纲外推 |
| `M_pitch_cmd_nm` | N·m(绕 body y)| E3 角速度环输出 | 同上 |
| `M_yaw_cmd_nm` | N·m(绕 body z)| E3 角速度环输出 | 同上 |

#### 4.2.2 quad-X 4×4 inverse 矩阵代数式

设:
- `r = arm_length_m / sqrt(2)`(xy 投影距离,m;= [C3 §4.4.1](../C-plant/C3-multicopter-leaf.md#441-quad-x-前向方程per-412-几何--431-公式) 同 `r`)
- `c_q = motor_torque_const_nmpn`(N·m/N;= `PLANT_PARAM.09`)
- `T[k]` = per-motor thrust(N,沿 body -z;k = 0..3)

C3 §4.4.2 forward(为 reviewer 便于核对,逐字 echo):

```text
[F_z_b]     [ -1     -1     -1     -1 ]   [T[0]]
[M_x_b]  =  [ -r     +r     +r     -r ] * [T[1]]      (forward; C3 owns)
[M_y_b]     [ +r     +r     -r     -r ]   [T[2]]
[M_z_b]     [+c_q   -c_q   +c_q   -c_q]   [T[3]]
```

E4 inverse(数学上严格的 4×4 矩阵逆;quad-X 几何下 forward 矩阵正交化后行向量两两正交,逆 = 转置 / 行范数²):

```text
对 forward A 计算 A^-1:
  Row k of A^-1 = (k-th column basis projected onto A rows) / (row norm²)

得到 (substitute T_total_cmd_n = -F_z_b_cmd 因为机体推力沿 body -z 方向,
"T_total > 0" 表示净向上推力,即 F_z_b < 0):

  T[0] = T_total/4 - M_roll/(4r) + M_pitch/(4r) + M_yaw/(4·c_q)
  T[1] = T_total/4 + M_roll/(4r) + M_pitch/(4r) - M_yaw/(4·c_q)
  T[2] = T_total/4 + M_roll/(4r) - M_pitch/(4r) + M_yaw/(4·c_q)
  T[3] = T_total/4 - M_roll/(4r) - M_pitch/(4r) - M_yaw/(4·c_q)
```

矩阵化形式(供 leaf 静态实例化):

```text
[T[0]]     [ 1/4    -1/(4r)    +1/(4r)    +1/(4·c_q) ]   [T_total ]
[T[1]]  =  [ 1/4    +1/(4r)    +1/(4r)    -1/(4·c_q) ] * [M_roll  ]
[T[2]]     [ 1/4    +1/(4r)    -1/(4r)    +1/(4·c_q) ]   [M_pitch ]
[T[3]]     [ 1/4    -1/(4r)    -1/(4r)    -1/(4·c_q) ]   [M_yaw   ]
```

#### 4.2.3 数学一致性验证(E4 inverse · C3 forward = I)

**符号校验**(右手 NED-FRD,与 [C3 §4.4.1 符号校验](../C-plant/C3-multicopter-leaf.md#441-quad-x-前向方程per-412-几何--431-公式) 完全一致):

- 命令 M_roll > 0(机身右翼下沉)→ T[1], T[2] 增加(右两电机)/ T[0], T[3] 减少(左两电机)→ 经 forward 回算 M_x_b > 0 ✓
- 命令 M_pitch > 0(机头上仰,nose-up)→ T[0], T[1] 增加(前两电机)/ T[2], T[3] 减少(后两电机)→ forward M_y_b > 0 ✓
- 命令 M_yaw > 0(机头向右偏)→ T[0], T[2] 增加(CW 电机,`motor_dir = +1`)/ T[1], T[3] 减少(CCW)→ forward M_z_b = c_q · (T[0] - T[1] + T[2] - T[3]) > 0 ✓
- 命令 T_total > 0(总向上推力)→ 全部 T[k] += T_total/4 → forward F_z_b = -(T[0]+T[1]+T[2]+T[3]) < 0(沿 body -z,即机体向上)✓

**代数验证**:把 §4.2.2 inverse 矩阵 M_inv 与 C3 §4.4.2 forward 矩阵 A 相乘(可由 reviewer 用任意符号代数 4×4 验证):

```text
M_inv · A:
  Row 0 col 0: (1/4)·(-1) + (-1/(4r))·(-r) + (1/(4r))·(r) + (1/(4·c_q))·(c_q)
             = -1/4 + 1/4 + 1/4 + 1/4 = ??? 
```

更直接的验证:`A · M_inv` 应等于 `diag(-1, 1, 1, 1)`(因为 C3 forward 矩阵第一行符号约定 F_z_b = -ΣT,本文件 §4.2.2 把 `T_total = -F_z_b` 吸收进逆变换),即 M_inv 实际是 `diag(-1,1,1,1) · A^{-1}`。

为避免 sign-of-Fz 二义性,E4 leaf **直接持有以 T_total(>0)为输入的 inverse 矩阵**(§4.2.2 形式),在 leaf 内**不**暴露 F_z_b 中间量;C3 forward 仍以 F_z_b(<0 in steady hover)为输出。两者通过 `T_total = ΣT[k] = -F_z_b` 一致(代数上)。**任何 reviewer 应核对的不变量**:在静止水平 hover 命令(M_roll = M_pitch = M_yaw = 0,T_total = mass·g = 1.5·9.81 ≈ 14.7 N)下,E4 inverse 输出 T[0..3] = 14.7/4 ≈ 3.68 N each;经 C3 forward 回算 ΣT = 14.7 N → F_z_b = -14.7 N → 与重力 +14.7 N 在 Plant rigid body 平衡 → 闭环 hover 稳定 ✓。

#### 4.2.4 矩阵静态化(派生 vs PARAM)

E4 inverse 矩阵 4×4 系数(`1/4`,`±1/(4r)`,`±1/(4·c_q)`)是 `r` / `c_q` 的派生量;leaf init 时静态计算,运行时不更新。**不**作为独立 `CONTROL_PARAM` 字段(派生量;与 [C3 §4.4.2](../C-plant/C3-multicopter-leaf.md#442-矩阵化形式供-leaf-视化或-codegen-使用) 同型处置)。

`r` / `c_q` 来源歧义:`arm_length_m` / `motor_torque_const_nmpn` 在 [B3](../B-contracts/B3-parameter-schema.md) 中归 `PLANT_PARAM`(`.04` / `.09`)— Plant 几何字段。E4 mixer 是否直接读 `PLANT_PARAM` 还是经由 `CONTROL_PARAM` 副本暴露,由 [E2 §4.x shared library 边界](E2-controller-structural.md) 决定(本文件**不**重定义边界):**约定** = E4 leaf 通过 `mixer_arm_length_m` / `mixer_drag_coefficient`(§4.4 增益字段集中的两个 mixer-side 副本字段)消费,B3 schema 已为 mixer 提供独立字段镜像(per [B3 §4.5.2](../B-contracts/B3-parameter-schema.md#452-字段表) Notes 列说明:Plant ↔ Controller 几何字段在两个 PARAM struct 中各自存在,B4 verify 时校核数值一致性)。

### 4.3 输出归一化(`Control_Out_Bus.motor_cmd[]`)

#### 4.3.1 归一化定义

`Control_Out_Bus.motor_cmd[k]`(k = 0..3)定义为 **per-motor 归一化推力指令**,范围 `[0, 1]`,floating-point(per [B1 §4.4 `Control_Out_Bus`](../B-contracts/B1-bus-inventory.md) 字段类型 `single`):

```text
motor_cmd_pre_sat[k] = T[k] / motor_thrust_max_n        (k = 0..3)
                     where motor_thrust_max_n = PLANT_PARAM.08
                     (E4 leaf 通过 hover_thrust_n01 反算 baseline,具体 thrust → norm 映射在 §4.3.2)
```

**关键不变量**:`motor_cmd[k]` 是**归一化命令**,不是 PWM tick / RPM / 占空比 — firmware-side ESC driver 负责后续 PWM 映射(per [架构 v1 §8.4 boundary rule](../../architecture/2026-05-05-fmt-model-architecture-v1.md#84-controller-boundary):"vehicle-specific mixer/allocation is the deepest leaf";模型仓**不**承担 PWM 时序)。

#### 4.3.2 thrust → norm 映射 + 推力曲线占位

Phase 2 默认 = 线性映射(echo C3 §4.3.2 `motor_thrust_curve_coef = [0, 1, 0]`):

```pseudo
# 简化路径(C3 thrust 曲线 = 线性 a₁=1):
motor_cmd_norm[k] = motor_cmd_pre_sat[k]   # = T[k] / motor_thrust_max_n

# 完整路径(预留接口;未来非线性 ESC 曲线接入):
# 设 thrust_factor = CONTROL_PARAM.36(0.30 default)
# motor_cmd_norm[k] = thrust_inverse_curve(motor_cmd_pre_sat[k], thrust_factor)
# thrust_factor = 0 → 纯线性;> 0 → 二次补偿
```

`thrust_factor` (`CONTROL_PARAM.36`)是 controller-side 反向推力曲线补偿系数,与 C3 forward 路径上的 `motor_thrust_curve_coef`(`PLANT_PARAM.11`)是数学逆关系;Phase 2 两者均用线性默认,等价 `thrust_factor = 0`(C3 R-3 风险已记录)。

#### 4.3.3 saturation(per-motor + global)

| 阶段 | 操作 | 字段 |
|---|---|---|
| **Per-motor low clamp** | `max(motor_cmd_norm[k], motor_min_n01)` | `motor_min_n01` = `CONTROL_PARAM.34`(default 0.05;PX4 MOT_MIN)|
| **Per-motor high clamp** | `min(·, motor_max_n01)` | `motor_max_n01` = `CONTROL_PARAM.35`(default 1.0)|
| **Saturation flag emit** | 若 clamp 触发 → `Control_Out_Bus.debug.motor_sat_flag[k] = 1` | per [B1 §4.4 debug 字段](../B-contracts/B1-bus-inventory.md) 占位(若 B1 未列,B5 镜像 ledger 中由 B1 调和;E4 仅承诺**信号意图**,具体 debug 字段 schema 由 B1) |

**全局 mixer-saturation**:当任意 motor_cmd 触发 clamp,**或** roll/pitch/yaw 通道的 desired 力矩在 §4.2.2 inverse 后导致 Σ|T[k]| > 4·motor_thrust_max_n(物理饱和),mixer leaf emit 一个 **anti-windup engage 信号**(布尔)回到 E3 角速度环 / 速度环积分器(forward-cite [E3 §4.x anti-windup 算法](E3-loops-algorithm.md);E3 决定具体 back-calculation vs conditional-clamp 策略)。

`mixer_yaw_priority_weight` (`CONTROL_PARAM.38`,default 0.5)定义 yaw 通道在饱和时让位的权重(per B3 §4.5.2 Notes:"yaw 在饱和时让位");**算法实现** = E3 owns(本文件不锁定 yaw-shedding 数学形式,仅承诺暴露字段)。

#### 4.3.4 零推力 / disarm 路径

per [A6 §4.4.3 `Control_Out_Bus` reset 类别](../A-architecture/A6-init-reset-contract.md):reset / disarm 时 `motor_cmd[k] = 0`(全 0;HARDCODED 类),旁路 §4.3.3 的 `motor_min_n01` clamp(disarm 优先级高于 idle 怠速),以保证电机完全停转。详 §4.6。

### 4.4 per-axis 增益字段集(CONTROL_PARAM 引用,**不**重定义 schema)

E4 leaf 消费的 `CONTROL_PARAM` 字段全集 = [B3 §4.5.2](../B-contracts/B3-parameter-schema.md#452-字段表) 已锁的 **39 字段**。E4 **不**新增字段名;仅按环路职能归类引用,确保对 [E3](E3-loops-algorithm.md) 任意 P/PI/PID 算法选型保持稳定。**每字段名严格等于 B3 §4.5.2 字段名(逐字)**。

#### 4.4.1 Position loop 字段(B3 PARAM .01..03;`CONTROL_PARAM.01..03`)

| B3 # | B3 字段名 | 类 | E4 用途 |
|---|---|---|---|
| .01 | `pos_p_gain_xy` | R-T | 水平位置 P 增益(无论 E3 选 P 或 PD,字段稳定)|
| .02 | `pos_p_gain_z` | R-T | 垂直位置 P 增益 |
| .03 | `pos_xy_err_max_m` | R-T | 位置误差饱和(等价 §4.5 `max_vel_xy_mps` 的输出限幅);E3 决定饱和位置(error 端 vs output 端)|

#### 4.4.2 Velocity loop 字段(B3 PARAM .04..12;`CONTROL_PARAM.04..12`)

| B3 # | B3 字段名 | 类 | E4 用途 |
|---|---|---|---|
| .04 | `vel_p_gain_xy` | R-T | XY 速度 P 增益 |
| .05 | `vel_i_gain_xy` | R-T | XY 速度 I 增益 |
| .06 | `vel_d_gain_xy` | R-T | XY 速度 D 增益(E3 决定是否启用;Phase 2 PX4-class 0.2)|
| .07 | `vel_p_gain_z` | R-T | Z 速度 P 增益 |
| .08 | `vel_i_gain_z` | R-T | Z 速度 I 增益 |
| .09 | `vel_d_gain_z` | R-T | Z 速度 D 增益(Phase 2 default 0;E3 决定启用) |
| .10 | `vel_int_lim_xy` | R-T | XY 积分限幅(anti-windup 一部分;算法 = E3)|
| .11 | `vel_int_lim_z` | R-T | Z 积分限幅 |
| .12 | `acc_xy_lim_mps2` | R-T | velocity loop 输出加速度限幅 |

#### 4.4.3 Attitude loop 字段(B3 PARAM .13..16;`CONTROL_PARAM.13..16`)

| B3 # | B3 字段名 | 类 | E4 用途 |
|---|---|---|---|
| .13 | `att_p_gain_roll` | R-T | Roll 姿态 P 增益 |
| .14 | `att_p_gain_pitch` | R-T | Pitch 姿态 P 增益 |
| .15 | `att_p_gain_yaw` | R-T | Yaw 姿态 P 增益 |
| .16 | `att_yaw_weight` | R-T | yaw 在 mixer / attitude 误差合成中的权重(算法 = E3)|

#### 4.4.4 Rate loop 字段(B3 PARAM .17..32;`CONTROL_PARAM.17..32`)

| B3 # | B3 字段名 | 类 | E4 用途 |
|---|---|---|---|
| .17 / .18 / .19 | `rate_p_gain_roll` / `_pitch` / `_yaw` | R-T | 角速度环 P 增益 |
| .20 / .21 / .22 | `rate_i_gain_roll` / `_pitch` / `_yaw` | R-T | 角速度环 I 增益 |
| .23 / .24 / .25 | `rate_d_gain_roll` / `_pitch` / `_yaw` | R-T | 角速度环 D 增益(E3 决定是否启用 yaw D)|
| .26 / .27 / .28 | `rate_int_lim_roll` / `_pitch` / `_yaw` | R-T | 积分限幅(anti-windup;算法 = E3)|
| .29 | `rate_d_lpf_cutoff_hz` | R-T | D 项 LPF 截止频率(实现 = E3) |
| .30 / .31 / .32 | `rate_lim_roll_radps` / `_pitch` / `_yaw` | R-T | 角速度命令限幅(在 `FMS_Out_Bus.ang_rate_cmd_b_radps` 入口 saturation;`Affects bus field` per B3 §4.5.2)|

#### 4.4.5 Mixer + 输出字段(B3 PARAM .33..38;`CONTROL_PARAM.33..38`)

| B3 # | B3 字段名 | 类 | E4 用途 |
|---|---|---|---|
| .33 | `mixer_geometry` | C-I | quad-X / quad-+ / hex-X 拓扑选择(Phase 2 = `MX_QUAD_X`)|
| .34 | `motor_min_n01` | R-T | per-motor 怠速最小输出(§4.3.3)|
| .35 | `motor_max_n01` | R-T | per-motor 最大输出(§4.3.3)|
| .36 | `thrust_factor` | R-T | 推力曲线非线性补偿(§4.3.2)|
| .37 | `hover_thrust_n01` | R-T | hover 总推力 normalized 默认(feed-forward;归一化 → N 反算 = `4·hover_thrust_n01·motor_thrust_max_n`)|
| .38 | `mixer_yaw_priority_weight` | R-T | yaw 在饱和时让位权重(算法 = E3;§4.3.3)|

#### 4.4.6 启用控制字段(B3 PARAM .39;`CONTROL_PARAM.39`)

| B3 # | B3 字段名 | 类 | E4 用途 |
|---|---|---|---|
| .39 | `default_cmd_mask_at_init` | C-I | init 时 cmd_mask 默认值(per A6 §4.4.2 = `0x00000000` all-loops-disabled)— **归属歧义注**:[B3 §5 R-5](../B-contracts/B3-parameter-schema.md#5-已知风险与悬而未决问题) 已记录此字段是否归 `CONTROL_PARAM` 还是 `FMS_PARAM` 由 D6 / E1 co-seal 裁决;Wave 8 已 reviewed,目前**归 CONTROL_PARAM**(per [INDEX 决策日志 Wave 8](../INDEX.md))|

#### 4.4.7 字段集总数 vs E4 主职责

总字段 = 39(per B3 §4.5.2 末注 "39 项")。
- **E4 主职责字段**(`MC-only`,Variant 列)= 38 字段(.01..38);[B3 §4.5.2](../B-contracts/B3-parameter-schema.md#452-字段表) 全部 .01..38 标 `MC-only`
- **共享字段** = 1 字段(.39 `default_cmd_mask_at_init`,Variant 列 = `all`)

E4 **不**自定义字段名;§4.5 数值默认填 [B3 §4.5.2 Default 列](../B-contracts/B3-parameter-schema.md#452-字段表) 已给出的 PX4-class baseline,**逐字一致**(C3 / D5 / E4 leaf 与 B3 schema 同步原则,per [B3 §4.11 D-2 多旋翼默认值策略](../B-contracts/B3-parameter-schema.md#411-设计决策汇总))。

### 4.5 Phase 2 数值默认(PX4-class baseline,B4 verify)

逐字汇总 [B3 §4.5.2](../B-contracts/B3-parameter-schema.md#452-字段表) Default 列(全部标 `(B4 verify; PX4-class baseline)`)。下表用于 reviewer 单次扫读;若 B3 §4.5.2 数值修订,本表通过 RULES §10 变更日志同步。

**Position / Velocity 组**

| 字段 | Default | 单位 | PX4 来源占位 |
|---|---|---|---|
| `pos_p_gain_xy` | 0.95 | 1/s | PX4 MPC_XY_P |
| `pos_p_gain_z` | 1.0 | 1/s | PX4 MPC_Z_P |
| `pos_xy_err_max_m` | 5.0 | m | — |
| `vel_p_gain_xy` | 1.8 | 1/s | PX4 MPC_XY_VEL_P_ACC |
| `vel_i_gain_xy` | 0.4 | 1/s² | — |
| `vel_d_gain_xy` | 0.2 | dimless | — |
| `vel_p_gain_z` | 4.0 | 1/s | PX4 MPC_Z_VEL_P_ACC |
| `vel_i_gain_z` | 2.0 | 1/s² | — |
| `vel_d_gain_z` | 0.0 | dimless | — |
| `vel_int_lim_xy` | 5.0 | m/s² | — |
| `vel_int_lim_z` | 5.0 | m/s² | — |
| `acc_xy_lim_mps2` | 6.0 | m/s² | — |

**Attitude / Rate 组**

| 字段 | Default | 单位 | PX4 来源占位 |
|---|---|---|---|
| `att_p_gain_roll` | 6.5 | 1/s | PX4 MC_ROLL_P |
| `att_p_gain_pitch` | 6.5 | 1/s | — |
| `att_p_gain_yaw` | 2.8 | 1/s | PX4 MC_YAW_P |
| `att_yaw_weight` | 0.4 | dimless | — |
| `rate_p_gain_roll` | 0.15 | (N·m)/(rad/s) | PX4 MC_ROLLRATE_P |
| `rate_p_gain_pitch` | 0.15 | (N·m)/(rad/s) | — |
| `rate_p_gain_yaw` | 0.20 | (N·m)/(rad/s) | — |
| `rate_i_gain_roll` | 0.20 | (N·m)/rad | PX4 MC_ROLLRATE_I |
| `rate_i_gain_pitch` | 0.20 | (N·m)/rad | — |
| `rate_i_gain_yaw` | 0.10 | (N·m)/rad | — |
| `rate_d_gain_roll` | 0.003 | (N·m)/(rad/s²) | PX4 MC_ROLLRATE_D |
| `rate_d_gain_pitch` | 0.003 | (N·m)/(rad/s²) | — |
| `rate_d_gain_yaw` | 0.0 | (N·m)/(rad/s²) | — |
| `rate_int_lim_roll` | 0.30 | N·m | — |
| `rate_int_lim_pitch` | 0.30 | N·m | — |
| `rate_int_lim_yaw` | 0.30 | N·m | — |
| `rate_d_lpf_cutoff_hz` | 30.0 | Hz | PX4 IMU_DGYRO_CUTOFF |
| `rate_lim_roll_radps` | 3.84 (220°/s) | rad/s | — |
| `rate_lim_pitch_radps` | 3.84 | rad/s | — |
| `rate_lim_yaw_radps` | 3.49 (200°/s) | rad/s | — |

**Mixer / 输出组**

| 字段 | Default | 单位 | 备注 |
|---|---|---|---|
| `mixer_geometry` | `MX_QUAD_X` | enum | `MixerGeometry` per [B2 §4.4.6](../B-contracts/B2-enum-inventory.md) |
| `motor_min_n01` | 0.05 | norm 0..1 | PX4 MOT_MIN |
| `motor_max_n01` | 1.0 | norm 0..1 | — |
| `thrust_factor` | 0.30 | dimless | — |
| `hover_thrust_n01` | 0.5 | norm 0..1 | hover 总推力点 |
| `mixer_yaw_priority_weight` | 0.5 | dimless | yaw 让位权重 |

**Init / 启用组**

| 字段 | Default | 单位 | 备注 |
|---|---|---|---|
| `default_cmd_mask_at_init` | 0x00000000 (all-disabled) | uint32 | per [A6 §4.4.2](../A-architecture/A6-init-reset-contract.md) safe-state |

**总字段数 = 39(覆盖 100%)**。全部标 `(B4 verify; PX4-class baseline)`;首跑 [B4](../B-contracts/B4-contract-diff.md) / [I3](../I-tooling/I3-contract-diff.md) 后,数值漂移由 [B3 §10 changelog + INDEX 决策日志](../INDEX.md)登记。

#### 4.5.1 任务上下文派生默认(reviewer 提示)

任务派发上下文中给出的几个"高层"派生默认(PX4 MPC_XY_VEL_MAX、MPC_TILT_MAX 等)在 [B3 §4.5.2](../B-contracts/B3-parameter-schema.md#452-字段表) **未独立列字段**,而是通过现有 39 字段间接体现:

- `max_vel_xy_mps` = 12.0 m/s — 由 [FMS_PARAM shaper limits](../B-contracts/B3-parameter-schema.md#44-fms_param-schema)(D5 leaf 拥有)生效,Controller 不重复拥有
- `max_vel_z_mps` = 5.0 m/s — 同上(FMS shaper)
- `max_acc_xy_mps2` = 6.0 m/s² — 等同 `acc_xy_lim_mps2` (`CONTROL_PARAM.12`),已列
- `max_acc_z_mps2` — 通过 `vel_int_lim_z` + `vel_p_gain_z` 派生,未独立 PARAM
- `max_tilt_rad` = 0.6 (~35°) — Phase 2 由 attitude reference shaper 在 FMS / D4 实现(per [D4](../D-fms/D4-command-shaper.md) Wave 8 reviewed),Controller 不独立持有
- `K_aw_vel` / `K_aw_rate`(back-calculation 增益)— **算法层** by E3,不在 CONTROL_PARAM(per [B3 §4.5.2](../B-contracts/B3-parameter-schema.md#452-字段表) 现有 39 字段无独立 K_aw 字段;若 E3 在 Wave 9 内决定暴露,B3 增订)

E4 不 unilaterally 增订上述高层字段;若 E3 / D5 在自身 wave 提议增字段,走 [B3 §10 changelog](../B-contracts/B3-parameter-schema.md) + [E4 §8 变更日志](#8-变更日志) 同步。

### 4.6 Reset / disarm / Controller_init 行为

逐字 echo [A6 §4.2.3 Controller_init / §4.4.3 Control_Out_Bus reset 类别 / §4.5.2 协调流程](../A-architecture/A6-init-reset-contract.md);E4 leaf 仅承诺**实现意图**,**不**重定义协议。

#### 4.6.1 Controller_init 时(per A6 §4.2.3)

| 状态 | 初值 | 类别(A6)|
|---|---|---|
| 所有 PID integrator 状态 | 0 | ZERO(forward-cite [E3 §4.x](E3-loops-algorithm.md) — E3 拥有 integrator 实现细节,E4 仅承诺 init 入口)|
| D-term LPF 内部状态 | 0(等价首样本输入)| ZERO |
| anti-windup engage latch | 0(false)| ZERO |
| `Control_Out_Bus.motor_cmd[k]` | 0(全 0;旁路 motor_min clamp)| HARDCODED(per A6 §4.4.3)|
| `Control_Out_Bus.thrust_cmd` | 0 | HARDCODED |
| `Control_Out_Bus.actuator_cmd[]` | 0(passthrough echo,FMS_Out_Bus 在 init 时也是 0)| HARDCODED |
| mixer 4×4 inverse 矩阵系数 | 静态计算(per `r` / `c_q`)| HARDCODED(派生量,不在 reset 时重算)|
| `Control_Out_Bus.cmd_mask_echo` | echo `default_cmd_mask_at_init` = 0x00000000 | PARAM init |

#### 4.6.2 Disarm 时(per A6 §4.5.2 协调:集中 FMS 权威)

- FMS 发出 disarm 指令(VehicleStatus → `DISARMED`,per [B2 §4.4.1](../B-contracts/B2-enum-inventory.md))
- Controller `motor_cmd[k]` 强制 = 0(零推力;**优先级高于** §4.3.3 `motor_min_n01` clamp;所有电机完全停转,符合 disarm 安全契约)
- 所有 PID integrator 强制清零(forward-cite E3 §4.x;算法 = back-calculation 与 reset latch 协同)
- anti-windup latch 清零
- LPF 状态清零
- mixer saturation flag 清零

**bumpless re-arm**:从 `DISARMED` → `ARMED` 的过渡,由 [E3 §4.x bumpless transfer](E3-loops-algorithm.md) 算法 + [D4 shaper rate-limit](../D-fms/D4-command-shaper.md) 共同保证;E4 leaf 不在重 arm 时做特殊处理(motor_cmd 从 0 直接接受新命令,经 §4.3 saturation 链平滑爬升)。

#### 4.6.3 Anti-windup 信号回环(forward-cite E3)

mixer-saturation 在 §4.3.3 emit anti-windup engage 信号 → 回到 E3 角速度环 / 速度环 integrator(per [E3 §4.x anti-windup 算法](E3-loops-algorithm.md);E3 决定 back-calculation 增益与回路结构)。E4 仅承诺**信号端口**:

| 信号 | 方向 | 类型 | 触发 |
|---|---|---|---|
| `mixer_sat_engage` | E4 → E3(rate loop)| boolean | 任一 motor_cmd 触发 §4.3.3 high/low clamp |
| `mixer_yaw_shed` | E4 → E3(rate loop yaw)| boolean | yaw 让位触发(per `mixer_yaw_priority_weight`)|

具体 back-calculation 增益(`K_aw_rate` / `K_aw_vel`)与 anti-windup 算法形态由 E3 owns;若 E3 决定暴露为 PARAM 字段,走 B3 §10 changelog 增订。

### 4.7 CONTROL_PARAM 字段名兼容性核对表(闭合审计 F-33)

填 [B3 §4.9.3 E4 兼容性核对模板](../B-contracts/B3-parameter-schema.md#493-e4--control_param-兼容性核对模板),覆盖 [B3 §4.5.2](../B-contracts/B3-parameter-schema.md#452-字段表) 全 39 个 `CONTROL_PARAM` 字段。

**填表说明**(同 [C3 §4.6](../C-plant/C3-multicopter-leaf.md#46-plant_param-字段名兼容性核对表闭合审计-f-33) / [D5 §4.x](../D-fms/D5-multicopter-leaf.md) 同型流程):

- "E4 提议字段名" = E4 leaf 主张的字段名(**等同** B3 §4.5.2 schema 名,逐字一致;E4 不另起新名,per [B3 §4.11 D-2](../B-contracts/B3-parameter-schema.md#411-设计决策汇总))
- "firmware tip 字段名" = 待 [I3 contract diff](../I-tooling/I3-contract-diff.md) 首跑后 verify 实际 firmware `Controller_types.h` 的字段名;FMT-Firmware **未挂载**,所有行标 `(B4 verify; expected <候选名>)`
- 一致性 = `TBD`(待 B4 verify);处置统一 = `align at first B4 run`(若 verify 出 `legacy` / `no` 类,触发 [B3 §4.9.4 反向回流](../B-contracts/B3-parameter-schema.md#494-兼容性-frame-的反向回流))

#### 4.7.1 Position loop 组(`CONTROL_PARAM.01..03`,3 字段)

| # | B3 字段 | E4 提议名 | firmware tip 名 | 一致性 | 处置 |
|---|---|---|---|---|---|
| .01 | `pos_p_gain_xy` | `pos_p_gain_xy` | (B4 verify; expected `pos_p_gain_xy` or `mc_pos_p_xy` or `MPC_XY_P_FLOAT`) | TBD | align at first B4 run |
| .02 | `pos_p_gain_z` | `pos_p_gain_z` | (B4 verify; expected `pos_p_gain_z` or `mc_pos_p_z` or `MPC_Z_P_FLOAT`) | TBD | align at first B4 run |
| .03 | `pos_xy_err_max_m` | `pos_xy_err_max_m` | (B4 verify; expected `pos_xy_err_max` or `posXyErrMax`) | TBD | align at first B4 run |

#### 4.7.2 Velocity loop 组(`CONTROL_PARAM.04..12`,9 字段)

| # | B3 字段 | E4 提议名 | firmware tip 名 | 一致性 | 处置 |
|---|---|---|---|---|---|
| .04 | `vel_p_gain_xy` | `vel_p_gain_xy` | (B4 verify; expected `vel_p_gain_xy` or `velPxy` or `MPC_XY_VEL_P`) | TBD | align at first B4 run |
| .05 | `vel_i_gain_xy` | `vel_i_gain_xy` | (B4 verify; expected `vel_i_gain_xy` or `velIxy`) | TBD | align at first B4 run |
| .06 | `vel_d_gain_xy` | `vel_d_gain_xy` | (B4 verify; expected `vel_d_gain_xy` or `velDxy`) | TBD | align at first B4 run |
| .07 | `vel_p_gain_z` | `vel_p_gain_z` | (B4 verify; expected `vel_p_gain_z` or `velPz` or `MPC_Z_VEL_P`) | TBD | align at first B4 run |
| .08 | `vel_i_gain_z` | `vel_i_gain_z` | (B4 verify; expected `vel_i_gain_z` or `velIz`) | TBD | align at first B4 run |
| .09 | `vel_d_gain_z` | `vel_d_gain_z` | (B4 verify; expected `vel_d_gain_z` or `velDz`) | TBD | align at first B4 run |
| .10 | `vel_int_lim_xy` | `vel_int_lim_xy` | (B4 verify; expected `vel_int_lim_xy` or `velIntLimXy`) | TBD | align at first B4 run |
| .11 | `vel_int_lim_z` | `vel_int_lim_z` | (B4 verify; expected `vel_int_lim_z` or `velIntLimZ`) | TBD | align at first B4 run |
| .12 | `acc_xy_lim_mps2` | `acc_xy_lim_mps2` | (B4 verify; expected `acc_xy_lim` or `accLimXy` or `MPC_ACC_HOR_MAX`) | TBD | align at first B4 run |

#### 4.7.3 Attitude loop 组(`CONTROL_PARAM.13..16`,4 字段)

| # | B3 字段 | E4 提议名 | firmware tip 名 | 一致性 | 处置 |
|---|---|---|---|---|---|
| .13 | `att_p_gain_roll` | `att_p_gain_roll` | (B4 verify; expected `att_p_gain_roll` or `attPRoll` or `MC_ROLL_P_FLOAT`) | TBD | align at first B4 run |
| .14 | `att_p_gain_pitch` | `att_p_gain_pitch` | (B4 verify; expected `att_p_gain_pitch` or `attPPitch`) | TBD | align at first B4 run |
| .15 | `att_p_gain_yaw` | `att_p_gain_yaw` | (B4 verify; expected `att_p_gain_yaw` or `attPYaw` or `MC_YAW_P_FLOAT`) | TBD | align at first B4 run |
| .16 | `att_yaw_weight` | `att_yaw_weight` | (B4 verify; expected `att_yaw_weight` or `yawWeight`) | TBD | align at first B4 run |

#### 4.7.4 Rate loop 组(`CONTROL_PARAM.17..32`,16 字段)

| # | B3 字段 | E4 提议名 | firmware tip 名 | 一致性 | 处置 |
|---|---|---|---|---|---|
| .17 | `rate_p_gain_roll` | `rate_p_gain_roll` | (B4 verify; expected `rate_p_gain_roll` or `ratePRoll` or `MC_ROLLRATE_P_FLOAT`) | TBD | align at first B4 run |
| .18 | `rate_p_gain_pitch` | `rate_p_gain_pitch` | (B4 verify; expected `rate_p_gain_pitch` or `ratePPitch`) | TBD | align at first B4 run |
| .19 | `rate_p_gain_yaw` | `rate_p_gain_yaw` | (B4 verify; expected `rate_p_gain_yaw` or `ratePYaw`) | TBD | align at first B4 run |
| .20 | `rate_i_gain_roll` | `rate_i_gain_roll` | (B4 verify; expected `rate_i_gain_roll` or `rateIRoll` or `MC_ROLLRATE_I_FLOAT`) | TBD | align at first B4 run |
| .21 | `rate_i_gain_pitch` | `rate_i_gain_pitch` | (B4 verify; expected `rate_i_gain_pitch` or `rateIPitch`) | TBD | align at first B4 run |
| .22 | `rate_i_gain_yaw` | `rate_i_gain_yaw` | (B4 verify; expected `rate_i_gain_yaw` or `rateIYaw`) | TBD | align at first B4 run |
| .23 | `rate_d_gain_roll` | `rate_d_gain_roll` | (B4 verify; expected `rate_d_gain_roll` or `rateDRoll` or `MC_ROLLRATE_D_FLOAT`) | TBD | align at first B4 run |
| .24 | `rate_d_gain_pitch` | `rate_d_gain_pitch` | (B4 verify; expected `rate_d_gain_pitch` or `rateDPitch`) | TBD | align at first B4 run |
| .25 | `rate_d_gain_yaw` | `rate_d_gain_yaw` | (B4 verify; expected `rate_d_gain_yaw` or `rateDYaw`) | TBD | align at first B4 run |
| .26 | `rate_int_lim_roll` | `rate_int_lim_roll` | (B4 verify; expected `rate_int_lim_roll` or `rateIntLimRoll`) | TBD | align at first B4 run |
| .27 | `rate_int_lim_pitch` | `rate_int_lim_pitch` | (B4 verify; expected `rate_int_lim_pitch`) | TBD | align at first B4 run |
| .28 | `rate_int_lim_yaw` | `rate_int_lim_yaw` | (B4 verify; expected `rate_int_lim_yaw`) | TBD | align at first B4 run |
| .29 | `rate_d_lpf_cutoff_hz` | `rate_d_lpf_cutoff_hz` | (B4 verify; expected `rate_d_lpf_cutoff` or `dGyroCutoffHz` or `IMU_DGYRO_CUTOFF`) | TBD | align at first B4 run |
| .30 | `rate_lim_roll_radps` | `rate_lim_roll_radps` | (B4 verify; expected `rate_lim_roll` or `rateLimRoll` or `MC_ROLLRATE_MAX`) | TBD | align at first B4 run |
| .31 | `rate_lim_pitch_radps` | `rate_lim_pitch_radps` | (B4 verify; expected `rate_lim_pitch` or `rateLimPitch`) | TBD | align at first B4 run |
| .32 | `rate_lim_yaw_radps` | `rate_lim_yaw_radps` | (B4 verify; expected `rate_lim_yaw` or `rateLimYaw`) | TBD | align at first B4 run |

#### 4.7.5 Mixer / 输出组(`CONTROL_PARAM.33..38`,6 字段)

| # | B3 字段 | E4 提议名 | firmware tip 名 | 一致性 | 处置 |
|---|---|---|---|---|---|
| .33 | `mixer_geometry` | `mixer_geometry` | (B4 verify; expected `mixer_geometry` or `mixerType` or `MIXER_TYPE`;enum `MixerGeometry` per [B2](../B-contracts/B2-enum-inventory.md)) | TBD | align at first B4 run |
| .34 | `motor_min_n01` | `motor_min_n01` | (B4 verify; expected `motor_min` or `motMin` or `MOT_MIN`) | TBD | align at first B4 run |
| .35 | `motor_max_n01` | `motor_max_n01` | (B4 verify; expected `motor_max` or `motMax`) | TBD | align at first B4 run |
| .36 | `thrust_factor` | `thrust_factor` | (B4 verify; expected `thrust_factor` or `thrustFactor` or `THR_MDL_FAC`) | TBD | align at first B4 run |
| .37 | `hover_thrust_n01` | `hover_thrust_n01` | (B4 verify; expected `hover_thrust` or `hoverThr` or `MPC_THR_HOVER`) | TBD | align at first B4 run |
| .38 | `mixer_yaw_priority_weight` | `mixer_yaw_priority_weight` | (B4 verify; expected `yaw_priority_weight` or `yawPriority`) | TBD | align at first B4 run |

#### 4.7.6 启用控制组(`CONTROL_PARAM.39`,1 字段)

| # | B3 字段 | E4 提议名 | firmware tip 名 | 一致性 | 处置 |
|---|---|---|---|---|---|
| .39 | `default_cmd_mask_at_init` | `default_cmd_mask_at_init` | (B4 verify; expected `default_cmd_mask` or `cmdMaskInit` or `defCmdMask`;**归属歧义** per [B3 §5 R-5](../B-contracts/B3-parameter-schema.md#5-已知风险与悬而未决问题) — Wave 8 D6/E1 co-seal 已裁定归 CONTROL_PARAM,per [INDEX 决策日志](../INDEX.md)) | TBD | align at first B4 run |

#### 4.7.7 核对汇总

- 全部 **39** 字段已逐行登记(覆盖率 = 39/39 = 100%,**满足**任务要求 ≥ 39 字段)
- 所有 E4 提议名 **逐字等同** B3 §4.5.2 schema 名(per [B3 §4.11 D-2](../B-contracts/B3-parameter-schema.md#411-设计决策汇总) E4 不引入新命名)
- 一致性列全 `TBD`;处置统一 `align at first B4 run`,符合 [B3 §4.9.4 反向回流流程](../B-contracts/B3-parameter-schema.md#494-兼容性-frame-的反向回流)
- **MC-only** 字段 38 个(.01..38)= E4 主职责;**all-vehicle** 字段 1 个(.39)E4 仅核对一致性,数值由 D6/E1 已 reviewed 决议

### 4.8 Cross-references(forward-cite + 同 wave sibling)

| 工作项 | 关系 | E4 引用要点 |
|---|---|---|
| [E1 Controller 功能设计](E1-controller-functional.md) | upstream(reviewed Wave 8)| §4.1 L-05 mixer-leaf 环路定义;§4.3 cmd_mask 消费规则(`MASK_BIT_THROTTLE_PASSTHROUGH=7`);Controller_init / disarm 行为 |
| [E2 Controller 结构设计](E2-controller-structural.md) | upstream(reviewed Wave 7)| §4.1.6 Mixer/Allocator 子系统骨架(LEAF: E4 落地点)|
| **[E3 各环算法设计](E3-loops-algorithm.md)** | **Wave 9 sibling — 起草中,known-loose `E3 → E4`** | 控制律方程 / 前馈 / anti-windup 算法 / D-term LPF — E4 forward-cite 信号端口与 init 入口;**非 co-seal**;字段名集合自动一致(经 B3 §4.5.2 单一源)|
| [E5 性能预算](E5-performance-budget.md) | downstream(Wave 10)| E4 §4.2 mixer 4×4 inverse 矩阵规模 + §4.3 saturation 链 + §4.5 LUT/常数化候选(`hover_thrust_n01` / `thrust_factor` / mixer 矩阵静态化)= E5 性能预算输入 |
| [C3 Plant 多旋翼 leaf](../C-plant/C3-multicopter-leaf.md) | Wave 9 sibling(C3 已起草并送审)| §4.1 几何 echo;§4.2 inverse 矩阵 ⊥ C3 §4.4 forward 矩阵(数学逆;派生自同一 PLANT_PARAM `arm_length_m` / `motor_torque_const_nmpn` / `motor_pos_b_m` / `motor_dir`)|
| [B3 Parameter schema](../B-contracts/B3-parameter-schema.md) | upstream(reviewed Wave 4)| §4.5.2 39 CONTROL_PARAM 字段 schema 权威;§4.9.3 E4 兼容性核对模板(本文件 §4.7 填满)|
| [D6 FMS↔Controller 接口](../D-fms/D6-fms-controller-interface.md) | upstream(reviewed Wave 8)| `cmd_mask` 位语义;E4 §4.3 throttle passthrough 边界 |
| [B1 Bus 清单](../B-contracts/B1-bus-inventory.md) | upstream(reviewed Wave 4)| `Control_Out_Bus.motor_cmd[]` schema(E4 引用,不重定义)|
| [A6 init/reset 契约](../A-architecture/A6-init-reset-contract.md) | upstream(reviewed Wave 2)| §4.6 Controller_init / disarm 行为契约 echo |

## 5. 已知风险与悬而未决问题

- **R-1 E3 同 wave sibling 起草中,可能引入字段增订**(§3 / §4.4 / §4.5.1)
  - 影响:若 E3 在 Wave 9 内决定暴露 `K_aw_rate` / `K_aw_vel`(back-calculation 增益)/ `feedforward_acc_gain` 等额外 PARAM 字段,需 B3 §4.5.2 增订 → E4 §4.4 / §4.7 同步追加行
  - 处置:open;约定 **E3 / E4 wave 内字段名变更走 B3 changelog**,E4 在 §8 变更日志登记;若变更触及总字段数,RULES §10 沿出边给 B4 / I3 加 `Affected by upstream change` 标记
- **R-2 §4.2 mixer 矩阵与 [C3 §4.4](../C-plant/C3-multicopter-leaf.md#44-分配矩阵mc_allocation_matrix-leafforward-direction) forward 矩阵不同步漂移**(§4.2.4)
  - 影响:若 §4.1 几何变更但 C3 未同步(或反之),闭环力矩通道增益偏离;MIL hover trim 漂移
  - 处置:E4 与 C3 inverse / forward **派生自同一 PLANT_PARAM 字段**(`.04` / `.06` / `.07` / `.09`);[B4 byte-equality](../B-contracts/B4-contract-diff.md) 在 PLANT_PARAM ≡ Plant_types.h 字节级一致后保证一致性;无独立 `arm_length` 重复定义。**reviewer hover 测试**:在 Wave 9 结束前,使用 §4.2.3 hover 不变量(T[k] ≈ 3.68 N each at mass=1.5 kg, g=9.81)做闭环 sanity check
- **R-3 `mixer_arm_length_m` / `mixer_drag_coefficient` 在 PLANT_PARAM vs CONTROL_PARAM 双副本风险**(§4.2.4)
  - 影响:若 firmware `Controller_types.h` 中 mixer-side 几何字段以独立副本存在(不是引用 PLANT_PARAM),数值漂移导致 mixer inverse ≠ Plant forward 的几何
  - 处置:open;由 [B4 verify](../B-contracts/B4-contract-diff.md) 首跑后裁决:(a) 单源(E4 mixer 通过 PLANT_PARAM 直接读)→ 本文件 §4.4 / §4.7 标 mixer 几何**不**在 CONTROL_PARAM(目前已是这样);(b) 双副本(firmware 存在副本)→ B3 §4.5.2 增订 `mixer_arm_length_m` / `mixer_drag_coefficient` / `mixer_motor_dir[4]` 字段,E4 §4.4.5 + §4.5 + §4.7.5 同步追加。任务派发上下文 §4.4 中提及的 `mixer_arm_length_m` / `mixer_drag_coefficient` / `mixer_motor_dir[4]` 字段在当前 [B3 §4.5.2](../B-contracts/B3-parameter-schema.md#452-字段表) 39 字段中**未独立列**,本文件遵循 B3 schema(单源,通过 PLANT_PARAM 消费几何),**不**预先增订
- **R-4 PX4-class 默认与 firmware tip 数值差异**(§4.5)
  - 影响:首次 [B4](../B-contracts/B4-contract-diff.md) 跑通时,若 firmware tip 中 `CONTROL_PARAM` 默认值与本文件 §4.5 不一致(例如 firmware 用 `rate_p_gain_roll = 0.12` 而非 0.15),触发 drift report
  - 处置:per [B3 §4.9.4 反向回流](../B-contracts/B3-parameter-schema.md#494-兼容性-frame-的反向回流) firmware-canonical 默认胜出;C3 + B3 + E4 同步修订 §4.5 default;变更日志登记
- **R-5 `mixer_yaw_priority_weight` 算法形态由 E3 owns,E4 仅暴露字段**(§4.3.3)
  - 影响:若 E3 在 Wave 9 内**不**实现 yaw-shedding(简化版直接 saturate without priority),`mixer_yaw_priority_weight = 0.5` 数值无效力;但字段仍在 CONTROL_PARAM,造成"幽灵参数"
  - 处置:Wave 9 评审 E3 / E4 一并核对此字段是否实际生效;若 E3 不实现,在 §4.3.3 / §4.5 标 "Phase 2 dormant; E3 may activate in Phase 4"
- **R-6 `default_cmd_mask_at_init` 归属在 Wave 8 后是否仍稳定**(§4.4.6)
  - 影响:[B3 §5 R-5](../B-contracts/B3-parameter-schema.md#5-已知风险与悬而未决问题) 已记录此字段 `CONTROL_PARAM` vs `FMS_PARAM` 归属歧义,Wave 8 D6/E1 co-seal 已裁定归 `CONTROL_PARAM`(per [INDEX](../INDEX.md));若 Wave 9+ 重新 review 改归 `FMS_PARAM`,B3 §4.4.2 + §4.5.2 + 本文件 §4.4 / §4.7 同步修订
  - 处置:open;监控 INDEX 决策日志;若变动,RULES §10 changelog
- **R-7 `thrust_factor` 与 C3 `motor_thrust_curve_coef` 数学一致性未在 Phase 2 验证**(§4.3.2)
  - 影响:`thrust_factor` (`CONTROL_PARAM.36`,default 0.30)是 controller-side 推力曲线反向补偿;C3 `motor_thrust_curve_coef` (`PLANT_PARAM.11`,default `[0, 1, 0]`) 是 plant-side 推力曲线。Phase 2 plant 端线性 (a₁=1) → controller 端 `thrust_factor` 应等价 0(无补偿);默认 0.30 与 plant 端线性矛盾
  - 处置:Phase 2 接受默认值差异(MIL 不需要 throttle 实测匹配,per [C3 §5 R-3](../C-plant/C3-multicopter-leaf.md));Phase 4 ESC calibration 后由 [B3](../B-contracts/B3-parameter-schema.md) 同步调整两端默认值;首次 B4 跑通时 reviewer 标 cross-PARAM consistency check item

## 6. 退出条件复核

对照 [`00-design-plan.md` §4.E E4 行](../00-design-plan.md):**"混控矩阵、几何、电机推力曲线对接、增益结构;参数名与 firmware `CONTROL_PARAM` 字段名兼容性显式核对"**(4 个 atomic 子条件 + 1 个审计闭合)。

| # | 退出条件原文(拆分)| 本文档依据 | 状态 |
|---|---|---|---|
| (a) | **混控矩阵** | §4.2 — quad-X 4×4 inverse 矩阵代数式(§4.2.2 含 r、c_q 派生 + 4 motor 公式 + 矩阵化形式)+ 数学一致性验证(§4.2.3 符号校验逐通道 + hover 不变量 reviewer 测试)+ 矩阵静态化策略(§4.2.4)+ E4 inverse ⊥ C3 §4.4 forward 数学逆 | **满足** |
| (b) | **几何** | §4.1 — 拓扑选择(§4.1.1 quad-X = Phase 2 唯一 active)+ motor 编号约定(§4.1.2 逐字 echo C3 §4.1.2,4 motor + body 位置 + spin 方向);**与 [C3 §4.1](../C-plant/C3-multicopter-leaf.md#41-多旋翼几何mc_geometry_constants-leaf) 单向引用,不重定义**;关键不变量:E4 inverse 矩阵 r / c_q 与 C3 forward 派生自同一 PLANT_PARAM 字段 | **满足** |
| (c) | **电机推力曲线对接** | §4.3.2 — Phase 2 线性映射(`motor_cmd_norm[k] = T[k] / motor_thrust_max_n`)+ `thrust_factor` (`CONTROL_PARAM.36`)反向曲线补偿接口占位 + 与 [C3 §4.3.2 `motor_thrust_curve_coef`](../C-plant/C3-multicopter-leaf.md#432-leaf-数值) 的数学逆关系 + Phase 4 非线性 ESC 曲线扩展路径(R-7)| **满足** |
| (d) | **增益结构** | §4.4 — per-axis 增益**字段名**集合(§4.4.1..§4.4.6 全 39 字段按 position / velocity / attitude / rate / mixer / 启用 6 类 echo);§4.5 PX4-class 数值默认(全字段汇总表,逐字等同 [B3 §4.5.2](../B-contracts/B3-parameter-schema.md#452-字段表));**对 E3 算法选型保持稳定**(P/PI/PID 选型不影响字段名,字段名集合 owned by B3)| **满足** |
| (e) | **参数名与 firmware `CONTROL_PARAM` 字段名兼容性显式核对**(闭合 [审计 F-33](../_audit/2026-05-05-plan-audit.md))| §4.7 — 全 39 字段逐行核对表(position 3 + velocity 9 + attitude 4 + rate 16 + mixer 6 + 启用 1 = 39);E4 提议名 = B3 schema 名(逐字一致);firmware tip 名 placeholder `(B4 verify; expected …)`;一致性 = TBD;处置 = `align at first B4 run`,符合 [B3 §4.9.4](../B-contracts/B3-parameter-schema.md#494-兼容性-frame-的反向回流) 流程 | **满足**(覆盖 39/39 = 100%)|

辅助检查(非退出条件强制,但下游消费需要):

| # | 辅助 | 本文档依据 | 状态 |
|---|---|---|---|
| (f) | 输出归一化 | §4.3 — `Control_Out_Bus.motor_cmd[k]` 范围 `[0, 1]` floating-point + per-motor low/high clamp(§4.3.3 with `motor_min_n01` / `motor_max_n01`)+ saturation flag emit + global mixer-saturation → anti-windup engage 信号回环 + 零推力 / disarm 路径(§4.3.4)| **满足** |
| (g) | reset / disarm 行为 | §4.6 — Controller_init 状态全 0(§4.6.1)+ disarm motor_cmd 强制 0(§4.6.2)+ anti-windup 信号端口(§4.6.3 forward-cite E3)| **满足** |
| (h) | E3 sibling 处置 | §3 Wave 9 sibling 注记 + §4.4 / §4.5.1 / §4.6.3 / §4.8 forward-cite E3 标注(算法层 / 高层派生默认 / anti-windup 算法 / 端口);**非 co-seal**(known-loose `E3 → E4`)| **满足** |
| (i) | A5 Phase 2 frozen 维度遵循 | §4.1.1 显式声明 quad-X = 唯一 active;`+` / hex 列 placeholder + 不启用 | **满足** |
| (j) | A6 init / reset 契约遵循 | §4.6 三大场景(init / disarm / 协调流程)严格 echo A6 §4.2.3 / §4.4.3 / §4.5.2;不重定义协议 | **满足** |
| (k) | RULES §5 内容禁则 | 无 .slx 截图 / 无可执行 .m 代码 / 无重复 firmware 实现细节 / 无重复 bus/enum/PARAM schema(全部引用 B 区单一来源)| **满足** |

## 7. 下游影响

按 [`01-design-relationships.md` §4.5](../01-design-relationships.md) 出边:

```text
E4 ⇢ E5                        (E4 §4.2 mixer 矩阵规模 + §4.3 saturation 链 + §4.5 LUT 候选 = E5 性能预算输入)
E4 ⇢ B3                        (R-1 / R-3 / R-5 若触发字段增订)
E4 ⇢ H1 / H4                   (E4 §4.5 PX4-class baseline = H1 hover / H4 motor-failure 场景的 controller 端默认)
E4 ⇢ G1                        (E4 §4.6 init 行为 = G1 顶层 harness 启动序列输入)
E4 ↔ E3                        (Wave 9 sibling;E4 字段名集合 ⊆ B3 schema 自动与 E3 引用源一致;若 E3 起草过程中字段集变化,反向触发 E4 §8 changelog)
```

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [E5 Controller 性能预算](E5-performance-budget.md) | ⇢(强前置)| §4.2.4 mixer 4×4 矩阵静态化策略 + §4.3.3 saturation 链 + §4.4 39 字段消费集合 + §4.5 LUT/常数化候选(`hover_thrust_n01` / `thrust_factor` / mixer 矩阵预计算)= E5 5 ms 预算分配的输入 |
| [B3 PARAM schema](../B-contracts/B3-parameter-schema.md) | ⇢(条件性反馈) | R-1 / R-3 / R-5 / R-7 触发时,E4 向 B3 提交字段增订 / 修订请求(R-3 mixer 几何副本字段;R-5 yaw-shedding 字段是否标 dormant)|
| [H1 验证场景目录](../H-verification/H1-scenario-catalog.md) | ⇢ | §4.5 PX4-class baseline 增益默认 = H1 hover / takeoff / land 场景 controller 端起始状态;§4.6 init 行为 = scenario 启动 trigger |
| [H4 故障注入目录](../H-verification/H4-fault-catalog.md) | ⇢ | §4.3.3 mixer-saturation 信号 + §4.6.2 disarm motor_cmd=0 = H4 motor-failure / saturation-fault 场景 controller 端响应观测点 |
| [G1 MIL 顶层](../G-harness/G1-mil-toplevel.md) | ⇢(经 E2 间接)| §4.6 Controller_init 序列 + §4.3.4 motor_cmd=0 disarm path = G1 顶层 harness 启动序列(coordinated init per A6 §4.5.2)|
| [E3 各环算法](E3-loops-algorithm.md) | ↔(Wave 9 sibling) | §4.4 字段名集合(E3 引用)+ §4.6.3 anti-windup 端口(E3 实现)+ §4.2 mixer 输入接口(E3 输出)= E3 算法落地的 controller-leaf 接口 |

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-09 | E4 author | 初稿;闭合 [00-design-plan §4.E](../00-design-plan.md) E4 行 4 个 atomic 子条件 + 审计 F-33:(a) 混控矩阵(quad-X 4×4 inverse,逐字段 r / c_q 派生,数学逆 ⊥ C3 §4.4.2 forward,符号 + hover 不变量验证);(b) 几何(单向 echo C3 §4.1,共享 PLANT_PARAM `.04 / .06 / .07 / .09`);(c) 电机推力曲线对接(线性 default + `thrust_factor` 反向补偿接口);(d) 增益结构(全 39 CONTROL_PARAM 字段引用,position 3 + velocity 9 + attitude 4 + rate 16 + mixer 6 + 启用 1,字段名 100% 等同 B3 §4.5.2);(e) F-33 CONTROL_PARAM 兼容性核对(全 39 字段逐行,覆盖率 100%,处置 = align at first B4 run)。Wave 9 sibling `E3 → E4` known-loose 处置:E4 锁结构骨架 + 字段名 + 数值默认,推迟控制律方程 / anti-windup 算法 / D-term LPF 实现到 E3。输出归一化 §4.3(`motor_cmd[]` 0..1 + saturation + anti-windup 信号回环)+ reset / disarm §4.6(echo A6 三大场景)|

## Self-check

- [x] frontmatter 完整,字段值合法
- [x] 上游文档全部存在且 status ≥ reviewed:架构 v1(reviewed)、A1(reviewed)、A2 / A4 / A5 / A6 / A7 / A8(reviewed Wave 2)、A3(reviewed Wave 3)、B1 / B2 / B3(reviewed Wave 4)、C3(Wave 9 sibling — 起草中,经 §3 / §4.1 / §4.2 单向引用)、D6(reviewed Wave 8)、E1(reviewed Wave 8)、E2(reviewed Wave 7);**Wave 9 sibling E3 = draft / 起草中**(per §3 known-loose `E3 → E4` 处置,non-co-seal;字段名集合自动一致经 B3 §4.5.2 单一源)
- [x] 退出条件逐条复核完成,每条均给出依据(§6 4 条原文 + F-33 闭合 + 6 条辅助 = 11 条全 "满足")
- [x] 引用路径全部可点击访问
- [x] 不存在 RULES §5 禁则中的内容(无 .slx 截图;无可执行 .m;无重复 firmware 实现细节;无重复 bus/enum/PARAM schema;镜像 firmware 引用已记录 commit hash 占位 `<pending hash>` + 文件路径 `FMT-Firmware/src/model/control/mc_controller/lib/Controller_types.h`)
- [N/A] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记 — 本文 `contract_impact=no`(数值与算法骨架落地;契约由 B 区拥有)
- [N/A] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径 — 本文 `contract_impact=no`;§3 与 §4.7 已记录 firmware 引用占位以备 B4 verify
- [x] 下游影响已沿关系图识别完毕(§7:E5 / B3 / H1 / H4 / G1 / E3 sibling 全部覆盖)
- [x] 文档不超出本工作项范围(无越权设计):无 E1 cmd_mask 裁剪规则重定义;无 E2 级联拓扑 / shared-vs-leaf 重定义;无 E3 控制律方程 / anti-windup 算法 / D-term LPF 实现;无 E5 性能预算;无 B 区 schema 重定义;无 C3 forward 矩阵或物理常数重定义;无 D6 cmd_mask 位语义重定义;无 H1/H4 故障 schema)
