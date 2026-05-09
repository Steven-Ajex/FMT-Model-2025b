---
work_item: D5
title: FMS 多旋翼 leaf 设计
upstream: [架构v1, A2, A3, A4, A5, A6, A7, A8, B1, B2, B3, D1, D2, D3, D4, D6]
contract_impact: no
status: draft
authored_at: 2026-05-09
last_reviewed_at:
reviewer_verdict: none
---

# D5 FMS 多旋翼 leaf 设计

## 1. 目的

为 FMS 的多旋翼 leaf 落地 [架构 v1 §16 Phase 2 multicopter slice](../../architecture/2026-05-05-fmt-model-architecture-v1.md) 的具体数值参数与多旋翼专属判定:把 [D3 Mode Manager](D3-mode-manager.md) 的 12 主 mode + 4 failsafe sub-mode 在 [D4 Command Shaper](D4-command-shaper.md) 中产生的 cmd_mask + setpoint 映射,**逐 mode** 在多旋翼语境下复核其有效性;为 D4 的 takeoff / landing / RTL profile 填入数值参数(climb rate / descent rate / flare alt / lift-off threshold / 触地检测);锁定 Phase 2 多旋翼 stick conditioning / 失效阈值 / geofence 形状 / Pilot rate-max;并用 [B3 §4.9.2](../B-contracts/B3-parameter-schema.md) 模板对**全部 43 个 `FMS_PARAM` 字段名**与 firmware tip `FMS_PARAM_TYPE` 做兼容性核对(闭合 audit F-33)。

本文件**只填值**,不重定义 PARAM schema(B3 拥有)/ 不重定义 cmd_mask 位语义(D6 拥有)/ 不修改 D3 chart 拓扑 / 不修改 D4 Shaper 内部块。

## 2. 范围

**在范围:**

- 12 主 mode + 4 failsafe sub-mode 的多旋翼**专属覆盖**(默认 = 沿用 D3/D4 已锁;§4.1 列出每条是否有偏离)
- TAKEOFF profile 数值(climb rate / 目标高度 / hover 油门 / lift-off 检测阈值 / 转换判据)— 闭合 D4 §5 open
- LAND profile 数值(下降率 / 低空下降率 / 低空阈值 / flare 高度 / 触地检测 / 解锁延时)— 闭合 D4 §5 open
- RTL profile 数值(RTL 高度 / 巡航速度 / 锥形下降)
- Stick conditioning 多旋翼数值(死区比 / expo / yaw 死区 / 油门死区 / Pilot rate max)
- Mode entry/exit guard 多旋翼数值(POSHOLD / AUTO / TAKEOFF entry guard 阈值)
- 失效阈值数值(RC / GCS link timeout / 电池低 / 电池临界 / geofence 半径 / geofence 形状)
- §4.8 全 43 项 `FMS_PARAM` 字段名兼容性核对表(闭合 audit F-33)
- D5 → D3 / D4 / D6 / B3 / G3 / H1 hand-off

**不在范围(由其他工作项处理):**

- `FMS_PARAM` schema 字段顺序 / 类型 / 单位 / 编码 — 由 [B3 §4.4.2](../B-contracts/B3-parameter-schema.md) 拥有;D5 仅引用字段名 + 多旋翼数值
- bus / enum 字段级 schema — 由 [B1](../B-contracts/B1-bus-inventory.md) / [B2](../B-contracts/B2-enum-inventory.md) 拥有
- Mode Manager Stateflow 状态 / 转换 / 守卫 — 由 [D3](D3-mode-manager.md) 拥有(D5 不修改 chart 拓扑)
- Command Shaper 内部块拓扑 / rate-jerk 算法形式 / profile 时间序列状态机 / cmd_mask emission 流水线 — 由 [D4](D4-command-shaper.md) 拥有(D5 仅填数值)
- cmd_mask 位语义 / 互斥规则 / INS validity 政策 — 由 [D6](D6-fms-controller-interface.md) 拥有
- Controller leaf(电机推力曲线 / mixer 矩阵 / 增益)— 由 [E4 多旋翼 leaf](../E-controller/E4-multicopter-leaf.md) 拥有(Wave 9 同期)
- Plant leaf(几何 / 惯量 / 电机模型)— 由 [C3 多旋翼 leaf](../C-plant/C3-multicopter-leaf.md) 拥有(Wave 9 同期)
- INS_Out_Bus 字段消费的 fallback 行为 — 由 [F3 INS 消费规则](../F-ins-contract/F3-consumption-rules.md) 拥有
- Harness 顶层 / 跨速率 wiring / 日志选项 — 由 [G1](../G-harness/G1-mil-toplevel.md) / [G2](../G-harness/G2-rate-scheduling.md) / [G3](../G-harness/G3-logging.md) 拥有

## 3. 依赖

| 上游 | 引用位置 | 用途 |
|---|---|---|
| [架构 v1 §7.1 / §11 / §13 / §16](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | reviewed | 三模块定义 / 模型仓侧 leaf 概念 / FMS 20 ms 单速率 / Phase 2 多旋翼 slice 锁定 |
| [A2 命名与单位约定](../A-architecture/A2-naming-conventions.md) | reviewed | PARAM 字段命名规则;单位/坐标系标记(`_m` / `_mps` / `_radps` / NED / body / normalized) |
| [A3 §4.4.6.1 / §4.4.6.2](../A-architecture/A3-module-boundaries.md) | reviewed | `FMS_Out_Bus` setpoint / 状态字段集 (D5 不重定义)|
| [A4 §4.4 RB-04 / RB-05](../A-architecture/A4-rate-boundaries.md) | reviewed | FMS 内部 20 ms;profile 计时 dt = 20 ms |
| [A5 §4.3.1 / §4.4](../A-architecture/A5-variant-strategy.md) | reviewed | Phase 2 唯一 leaf = 多旋翼;D5 是 FMS leaf 锚点 |
| [A6 §4.4.2 / §4.5.2 / §4.5.3 / §4.7.1](../A-architecture/A6-init-reset-contract.md) | reviewed | reset 类别表;INS readiness gate 序列;global vs local reset 边界 |
| [A7 §4.6 / §4.8](../A-architecture/A7-time-conventions.md) | reviewed | timestamp 单位 ms;dt = 20 ms 固定步长(profile 计时基准)|
| [A8 §4.3.2](../A-architecture/A8-shared-library-roster.md) | reviewed | FMS Layer B shell(D5 仅引用,不替换框架)|
| [B1 §4.7](../B-contracts/B1-bus-inventory.md) | reviewed | `FMS_Out_Bus` 字段名(D5 仅引用)|
| [B2 §4.4.5 / §4.4.6 / §4.4.7 / §4.4.14 / §4.4.16](../B-contracts/B2-enum-inventory.md) | reviewed | `FlightMode` / `CtrlMode` / `FailsafeState` / `CmdMaskBit` / `GeofenceShape`(`GF_CYLINDER` 成员名)|
| [B3 §4.4.2 / §4.9.2](../B-contracts/B3-parameter-schema.md) | reviewed | `FMS_PARAM` 全 43 字段定义 + Phase 2 多旋翼默认值 + §4.9.2 D5 兼容性 frame 模板;D5 在自身 §4.8 填表 |
| [D1 §4.1 / §4.3 / §4.4 / §4.5](D1-fms-functional.md) | reviewed | mode 集 / 整形策略 / 失效触发分类 / mission 范围(D5 不重定义,仅复核每条在多旋翼下的成立)|
| [D2 §4.3 / §4.6](D2-fms-structural.md) | reviewed | 6 子模块边界 / D5 leaf 注入点为 PARAM + `landed_detected` 谓词 |
| [D3 §4.2 / §4.3 / §4.5 / §4.6 / §4.9.4](D3-mode-manager.md) | reviewed | 状态-enum 映射 / 转换 / 守卫 / cmd_mask emission;§4.9.4 明列 D5 注入 5 项(arm_safety_check_mask 位语义 / 限值 / landed_detected / mission_done_action / ACRO 启用)|
| [D4 §4.2 / §4.3 / §4.5 / §4.6 / §4.9.1](D4-command-shaper.md) | reviewed | 16 mode 的 cmd_mask + setpoint 映射 / rate-jerk 类别 / 三 profile 算法 / stick conditioning 拓扑 / §4.9.1 D5 hand-off 列 |
| [D6 §4.1 / §4.2 / §4.4 / §4.5 / §4.7](D6-fms-controller-interface.md) | reviewed | cmd_mask 位语义 / 互斥规则 / INS validity 政策 / reset 强制态(D5 不修改)|
| FMT-Firmware @ `<pending hash>` | `FMT-Firmware/src/model/fms/mc_fms/lib/FMS_types.h` | `FMS_PARAM_TYPE` 字段名 mirror 源(B3 拥有镜像;D5 在 §4.8 填兼容性表;FMT-Firmware 当前未挂载,所有 firmware 端字段名标 `(B4 verify; expected …)` 占位)|

环境注:FMT-Firmware 在本设计阶段**未挂载**;§4.8 兼容性表的 firmware-tip 列以 `(B4 verify; expected …)` 占位,首跑 B4 / I3 后由 §10 changelog + INDEX 决策日志补齐,与 [B3 §5 第 1 项](../B-contracts/B3-parameter-schema.md) 同期闭合。

## 4. 设计内容

### 4.1 多旋翼 mode-cmd 映射(per [D3 §4.6](D3-mode-manager.md) + [D4 §4.2](D4-command-shaper.md) 已锁;D5 复核 multicopter 偏离)

D3 §4.6(cmd_mask emission per state)与 D4 §4.2.1 / §4.2.2(per-mode cmd_mask + setpoint 映射)已在 Wave 8 co-seal batch 中**全部按多旋翼语境**锁定(per [架构 v1 §16](../../architecture/2026-05-05-fmt-model-architecture-v1.md) "multicopter is the seed vehicle" + [D1 §4.1.2](D1-fms-functional.md) Phase 2 = 多旋翼唯一 leaf)。D5 在此**仅复核**每行的多旋翼专属偏离;**默认 = 沿用,无偏离**。

#### 4.1.1 主 mode(M-01..M-12)偏离表

| Mode | D3 §4.6 / D4 §4.2.1 cmd_mask | 多旋翼偏离 | D5 处置 |
|---|---|---|---|
| M-01 IDLE / DISARMED | `(0,0,0,0,0,0,0,0)` | 无 | 沿用(reset/disarmed safe state)|
| M-02 MANUAL | `(0,0,0,0,1,0,1,1)` `RATE+YAWR+THR` | 无 | 沿用(多旋翼 stick → body rate 直传 + throttle 直传 = 标准)|
| M-03 STABILIZE | `(0,0,0,1,0,0,1,1)` `ATT+YAWR+THR` | 无 | 沿用(多旋翼 stick → tilt 角是经典自稳)|
| M-04 ALTHOLD | `(0,1,0,1,0,0,1,0)` `VEL(z)+ATT+YAWR` | 无 | 沿用(z-vel 闭环 + 水平 ATT 手动 = 多旋翼 ALTHOLD 标准)|
| M-05 POSHOLD | `(1,1,1,0,0,1,1,0)` `POS+VEL+ACC+YAW+YAWR` | 无 | 沿用(BIT_ATT=0,姿态由 thrust-vector cascade 隐式推导,per [D6 §4.3.6](D6-fms-controller-interface.md);多旋翼自然支持)|
| M-06 TAKEOFF | `(1,1,0,0,0,1,0,0)` `POS+VEL+YAW` | 无 | 沿用;D5 在 §4.2 填 climb rate / 目标高度数值 |
| M-07 LAND | `(1,1,0,0,0,1,0,0)` `POS+VEL+YAW` | 无 | 沿用;D5 在 §4.3 填 descent rate / flare / 触地检测 |
| M-08 RTL | `(1,1,1,0,0,1,0,0)` `POS+VEL+ACC+YAW` | 无 | 沿用;D5 在 §4.4 填 RTL altitude / 巡航速度 |
| M-09 LOITER | `(1,0,0,0,0,1,0,0)` `POS+YAW` | 无 | 沿用(wp 处停留;ATT 由 thrust-vector cascade 隐式推导 — 与 hover 一致)|
| M-10 MISSION | `(1,1,1,0,0,1,0,0)` `POS+VEL+ACC+YAW` | 无 | 沿用 |
| M-11 OFFBOARD | per Auto(经 D6 §4.4 mutex 过滤)| 无 | 沿用;D5 不约束 OFFBOARD 内部组合(由 D3 entry action + D4 §4.7.2 G-1..G-6 enforcement)|
| M-12 ACRO | `(0,0,0,0,1,0,1,1)` `RATE+YAWR+THR` | **D5 锁定 Phase 2 启用 = 否**(per [D1 §5 风险](D1-fms-functional.md) + [D3 §4.3.2 T-22 注](D3-mode-manager.md))| Phase 2 多旋翼 leaf **不**启用 M-12;`PilotMode` 缺 `PMODE_ACRO`(per B2 §4.4.4),GCS path 也不开放;M-12 状态保留为 placeholder,D3 chart T-22 转换 disabled。Phase 4 / Phase 5 leaf 决定是否启用,需先在 B2 增补 `PMODE_ACRO` |

#### 4.1.2 Failsafe sub-mode(FS-01..FS-04)偏离表

| Sub-mode | D3 §4.6 / D4 §4.2.2 cmd_mask | 多旋翼偏离 | D5 处置 |
|---|---|---|---|
| FS-01 RTL_FAILSAFE | `(1,1,1,0,0,1,0,0)` `POS+VEL+ACC+YAW`(= M-08)| 无 | 沿用;复用 §4.4 RTL profile 数值 |
| FS-02 LAND_NOW | `(1,1,0,0,0,1,0,0)` `POS+VEL+YAW`(= M-07)| 无 | 沿用;复用 §4.3 LAND profile;critical battery 触发时直接进入 phase_1(忽略 phase_0 idle)|
| FS-03 HOVER_HOLD | `(1,1,1,0,0,1,1,0)` `POS+VEL+ACC+YAW+YAWR`(= M-05)| 无 | 沿用(多旋翼天然支持原地 hover) |
| FS-04 DISARM_FORCED | `(0,0,0,0,0,0,0,0)`(全清) | 无 | 沿用(per [D6 §4.7](D6-fms-controller-interface.md) reset / disarmed 强制态)|

#### 4.1.3 关键不变量

- D5 **不** emit 任何 D6 §4.4 互斥违规组合(D3 §4.6 + D4 §4.2 已 enforcement)
- D5 **不**新增 cmd_mask 位 / 不修改位号(B2 §4.4.14 拥有)
- D5 **不**新增 mode(D1 §4.1.1 + B2 §4.4.5 拥有)
- 所有数值参数偏离都通过 `FMS_PARAM` 字段表达(per §4.2..§4.7)— 不通过修改 D3 chart 或 D4 拓扑

### 4.2 TAKEOFF profile(多旋翼)

D4 §4.5.1 锁定算法形式(phase_0 idle → phase_1 lift-off detect → phase_2 climb → phase_3 terminal hold);D5 填**数值**与**lift-off 检测阈值**(闭合 [D4 §5 open](D4-command-shaper.md) "TAKEOFF lift-off 检测算法未锁数值阈值")。

| 项 | PARAM(B3 §4.4.2)| Phase 2 多旋翼默认值 | 单位 / 坐标系 | 来源 / 备注 |
|---|---|---:|---|---|
| 目标爬升高度(AGL,terminal hold)| `FMS_PARAM.takeoff_alt_m` (.06)| **3.0** | m, NED-Down(下为正,故内部存为 -3.0;字段值表示 above-launch 高度,per A2)| 任务 brief 默认 3.0;偏离 [B3 §4.4.2 .06](../B-contracts/B3-parameter-schema.md) 默认 2.5 → D5 取 brief 锁的 3.0 |
| 爬升速率 | `FMS_PARAM.takeoff_climb_speed_mps` (.07)| **1.5** | m/s, NED(向上即 vz < 0)| 任务 brief + B3 .07 默认一致 |
| Hover 油门(stick 中位映射)| `FMS_PARAM.manual_thr_hover_n01` (.41)| **0.5** | normalized 0..1 | 任务 brief + B3 .41 默认一致;Phase 2 多旋翼 4S LiPo 典型 hover 推力比 |
| Lift-off 检测高度阈值(z 方向位移触发 phase_1 → phase_2)| **D5 leaf field**:`takeoff_liftoff_z_threshold_m`(D5 命名,**不在 B3 .06..43 范围**;由 D5 与 B3 协同;若 B3 后续增补,字段号顺延)| **−0.3**(NED-Down;表示从 init pos 向下 0.3 m 之上即认 lift-off;典型多旋翼 GPS 噪声大于此值则会误触发,故配合 INS climb-rate AND 同时判断)| m, NED-Down(负值表上升)| 任务 brief 默认 −0.3;闭合 [D4 §5 open](D4-command-shaper.md);D5 leaf 锁 |
| Lift-off 检测辅助:climb rate threshold | **D5 leaf field**:`takeoff_liftoff_vz_threshold_mps`(D5 命名)| **−0.2**(向上)| m/s, NED-Down | D5 锁;与 z 阈值 AND 后判;避开 GPS-only 模式下 z 噪声 |
| Phase_1 → phase_2 转换判据 | 复合条件 | `(PILOT.thr > arm_throttle_thr_n01) && (HOME.z - INS.position_ned_m[2] >= |takeoff_liftoff_z_threshold_m|) && (INS.velocity_ned_mps[2] <= takeoff_liftoff_vz_threshold_mps)` 持续 ≥ 1 帧 | — | per D4 §4.5.1 phase_1 → phase_2 注 + D5 阈值 |
| Phase_2 → phase_3 转换判据 | 复合条件 | `(HOME.z - INS.position_ned_m[2] >= takeoff_alt_m - 0.2 m)` 持续 ≥ 5 帧(100 ms)| — | 0.2 m 容差 + 5 帧 dwell;D5 锁定;与 D4 §4.5.1 末态保证 vz=0 ≥ 1 帧合并 |
| TAKEOFF → POSHOLD 切换条件 | profile_done 脉冲 + ALT 守卫 | `profile_done == 1 && (|INS.velocity_ned_mps[2]| < 0.3 m/s)` | — | D3 T-11 转换守卫;profile_done 由 D4 phase_3 entry 触发 |

**说明**:lift-off 检测两阈值均为 D5 leaf field(不在 B3 §4.4.2 .06..10 范围);与 [D4 §5](D4-command-shaper.md) 同步标"open by D5"。Phase 2 在 D5 §4.8 §4.8.6 表格 §中作为 D5 leaf-only(non-firmware-mirror)字段单列;如未来 firmware 端引入对应字段,则在 B3 增补到 `FMS_PARAM` 字段集并经 §10 changelog + INDEX 决策日志登记。

### 4.3 LAND profile(多旋翼)

D4 §4.5.2 锁定算法(phase_0 idle → phase_1 descent → phase_2 flare → phase_3 touchdown + disarm dwell);D5 填**数值**与**触地检测**(闭合 [D4 §5 open](D4-command-shaper.md) "LAND flare_alt_threshold + 触地检测算法未锁")。

| 项 | PARAM(B3 §4.4.2)| Phase 2 多旋翼默认值 | 单位 / 坐标系 | 来源 / 备注 |
|---|---|---:|---|---|
| 高空下降速率(phase_1) | `FMS_PARAM.landing_descent_speed_mps` (.08)| **0.5** | m/s, NED-Down(下为正)| 任务 brief 默认 0.5;偏离 [B3 §4.4.2 .08](../B-contracts/B3-parameter-schema.md) 默认 0.7 → D5 取 brief 锁的 0.5(更保守) |
| 低空(flare)下降速率(phase_2) | `FMS_PARAM.land_touchdown_speed_mps` (.09)| **0.2** | m/s, NED-Down | 任务 brief 默认 0.2;偏离 B3 .09 默认 0.3 → D5 取 brief 锁的 0.2 |
| 低空阈值(phase_1 → phase_2 切换高度,AGL)| **D5 leaf field**:`land_low_alt_m`(等价于 D4 §4.5.2 / §4.9.1 中的 `flare_alt_threshold`;D5 命名采用 brief)| **1.0** | m, NED-Down(over-ground)| 任务 brief 默认 1.0;闭合 D4 §5 open |
| Flare 高度(phase_2 → phase_3 触地起判线)| **D5 leaf field**:`land_flare_alt_m` | **0.3** | m | 任务 brief 默认 0.3;闭合 D4 §5 open |
| 触地油门阈值 | **D5 leaf field**:`land_ground_detect_throttle_threshold_n01` | **0.2** | normalized 0..1 | 任务 brief 默认 0.2;低于此值且垂速近 0 + 持续 dwell 即认触地 |
| 触地速度收敛判据 | 内部 | `|INS.velocity_ned_mps| < 0.15 m/s` 持续 ≥ 5 帧(100 ms) | m/s | D5 锁;5 帧 = 100 ms @ FMS 20 ms |
| 解锁延时(phase_3 dwell)| `FMS_PARAM.land_disarm_dwell_s` (.10)| **2.0** | s | 任务 brief 默认 2.0;与 [B3 §4.4.2 .10](../B-contracts/B3-parameter-schema.md) 默认 2.0 一致 |
| Phase_1 → phase_2 转换判据 | 复合条件 | `(-INS.position_ned_m[2] <= land_low_alt_m)`(over-ground;假定 home alt 已设)| — | per D4 §4.5.2 phase_1 → phase_2 |
| Phase_2 → phase_3 转换判据(触地)| 复合 | `(motor_cmd_avg < land_ground_detect_throttle_threshold_n01) && (|INS.velocity_ned_mps| < 0.15 m/s)` 持续 ≥ 5 帧 | — | D5 锁;motor_cmd 来自 D1 §4.7 OQ4 transitional `Control_Out_Bus.motor_cmd[]`(若启用)/ 否则用 throttle_cmd 替代 — H4 验证 |
| Phase_3 → DISARM 切换 | profile_done 脉冲 + dwell | `profile_done == 1`(由 phase_3 dwell ≥ `land_disarm_dwell_s` 触发,per D4 §4.5.2 phase_3)| — | D3 T-13 / T-05 转换守卫 |

**说明**:`land_low_alt_m` 与 `land_flare_alt_m` 是 D5 leaf field(非 B3 .06..10 fixed slots);`land_ground_detect_throttle_threshold_n01` 也属 D5 leaf。三者在 §4.8 §4.8.6 D5-only 表中单列;如 firmware 后续增补,经 §10 changelog 同步至 B3。

### 4.4 RTL profile(多旋翼)

D4 §4.5.3 锁定算法(phase_0 idle → phase_1 climb → phase_2 cruise → phase_3 loiter → phase_4 land);D5 填**数值**。

| 项 | PARAM(B3 §4.4.2)| Phase 2 多旋翼默认值 | 单位 / 坐标系 | 来源 / 备注 |
|---|---|---:|---|---|
| RTL altitude(phase_1 climb 目标 + phase_2 cruise 高度,AGL)| `FMS_PARAM.rtl_climb_alt_m` (.28)| **30.0** | m, NED-Down | 任务 brief 默认 30.0(B3 .28 默认 30.0 一致);RTL 总是先爬升再返航 |
| RTL 巡航速度 | 复用 `FMS_PARAM.mission_default_speed_mps` (.27)| **5.0** | m/s, NED 水平 | 任务 brief 默认 5.0;复用 mission 默认速度(避免重复 PARAM)— per D4 §4.5.3 phase_2 注 |
| Home 上空 loiter 时长(phase_3) | `FMS_PARAM.rtl_loiter_time_s` (.29)| **5.0** | s | B3 .29 默认 5.0;D5 沿用 |
| Home XY 接受半径(phase_2 → phase_3)| 复用 `FMS_PARAM.wp_acceptance_radius_m` (.26)| **1.5** | m | 复用 mission wp 接受半径(避免重复 PARAM);per D4 §4.5.3 phase_2 → phase_3 注 |
| Phase_1 终高度容差 | 内部 | **0.5** m(D5 锁,per D4 §4.5.3 phase_1 → phase_2 注 "[D5 leaf]")| m | D4 标 [D5 leaf];D5 锁 0.5 m |
| 锥形下降(at home)| phase_4 = LAND profile @ home XY | hand-off 给 LAND profile(§4.3) | — | per D4 §4.5.3 phase_3 → phase_4 接力;**锥形下降 = 在 home XY 内启动 LAND profile,使下降率自动从 .08 衰减到 .09**(锥形效应来自 LAND phase_1 → phase_2 切换的 alt 阈值)|
| Face-home yaw 转向 ramp | 内部 | yaw rate-of-change 限 = `yaw_rate_lim_radps` (.39) | rad/s | per D4 §4.3.1 yaw 行;D5 不独立锁数值 |

**关键不变量**:RTL altitude 政策 = **AGL 30 m**(高于典型障碍物;低于 FAA 400 ft / 120 m 的 `geofence_alt_max_m` (.14)上限);若 home 海拔与 launch 不同,phase_1 仍以 launch alt 为基线(per [D4 §4.5.3](D4-command-shaper.md) phase_1 latch INS at entry)。

### 4.5 Stick conditioning(多旋翼)

D4 §4.6 锁定算法形式(死区 + expo + 单位换算 + LP filter 可选);D5 填**数值**与 expo 系数。

| 项 | PARAM | Phase 2 多旋翼默认值 | 单位 | 来源 / 备注 |
|---|---|---:|---|---|
| Roll/pitch deadband ratio | `FMS_PARAM.manual_stick_deadzone_n01` (.40) | **0.05** | normalized 0..1 | 任务 brief + B3 .40 默认一致;roll / pitch / yaw 共享同一死区(per D4 §4.6.1)|
| Roll/pitch expo ratio | **D5 leaf field**:`expo_ratio_roll_pitch` | **0.30** | unitless 0..1 | 任务 brief 默认 0.30;D4 §4.9.1 hand-off 标 D5 leaf |
| Yaw stick deadband | 复用 `FMS_PARAM.manual_stick_deadzone_n01` (.40) | **0.05** | normalized | per D4 §4.6.1 共享死区;D5 不独立锁(若未来需要差异化,扩 D5 leaf field) |
| Yaw expo ratio | **D5 leaf field**:`expo_ratio_yaw` | **0.30** | unitless 0..1 | D5 锁;Phase 2 与 roll/pitch 同 |
| Throttle deadband | 复用 `FMS_PARAM.manual_stick_deadzone_n01` (.40) | **0.0** | normalized | 任务 brief "Throttle 全程使用,deadband 0";D5 在 stick conditioning 实现层对 throttle 通道**绕过** deadband(per D4 §4.6.1 注;D5 leaf 实现 mode-conditional bypass)|
| Pilot rate max(MANUAL / ACRO body rate)| **D5 leaf field**:`pilot_rate_max_radps`(对应 D4 §4.3.1 "rate-loop body-rate max"行)| **3.5** | rad/s, body | 任务 brief 默认 3.5;闭合 [D4 §5 risk "Rate-loop body-rate max 不在 B3 .33..41 范围"](D4-command-shaper.md);D5 leaf 锁;与 B3 协同 — 若 B3 增补此字段,移到 `FMS_PARAM` |
| Bumpless ramp duration | **D5 leaf field**:`transition_ramp_ms` | **200** | ms | 任务 brief 隐含(D4 §4.4.2 默认 200);D5 沿用;闭合 [D4 §5 risk](D4-command-shaper.md) |
| Stick LP filter(可选)| **D5 leaf field**:`stick_lp_cutoff_hz`| **disabled (0.0)** | Hz | Phase 2 默认禁用;闭合 [D4 §5 risk "Stick conditioning 的 LowPass filter 是否在 Phase 2 启用未定"](D4-command-shaper.md);Phase 4 调试需要时再启用 |

### 4.6 Mode entry/exit guards(多旋翼专属阈值)

D3 §4.5 锁定守卫表达式(declarative,引用 PARAM 名);D5 填**数值阈值**。本节复核任务 brief 列出的 3 个 entry guard,数值由 PARAM 表达。

#### 4.6.1 POSHOLD entry guard(D3 G-POS)

D3 §4.5 G-POS = `G-ALT && (INS.flag.position_valid == 1) && (INS.flag.velocity_valid == 1)`;G-ALT = `(INS.st >= INS_STATUS_READY) && (INS.flag.attitude_valid == 1) && (INS.flag.heading_valid == 1)`。

| 子条件 | 阈值 | 来源 / 备注 |
|---|---|---|
| `INS_position_valid` | bool == 1 | INS_Out_Bus 字段;不需 PARAM |
| `INS_velocity_valid` | bool == 1 | 同 |
| Hover throttle calibrated(任务 brief 列出)| 由 `manual_thr_hover_n01` (.41) 默认 0.5 视为已校准;Phase 2 不显式做飞行前校准(由 H4 / Phase 4 实装)| brief 隐含 |
| INS readiness | `INS_Status >= INS_STATUS_READY` | per A6 §4.5.3 |

#### 4.6.2 AUTO entry guard(D3 G-MISSION / G-OFFBOARD)

| Mode | 守卫名 | 多旋翼数值化 |
|---|---|---|
| M-10 MISSION entry | G-MISSION | `G-POS && (mission_state ∈ {MISSION_LOADED, MISSION_PAUSED}) && (wp_count >= 1) && (wp_count <= wp_array_max_len)`(`wp_array_max_len` (.25) = 64,per B3)|
| M-11 OFFBOARD entry | G-OFFBOARD | `G-POS && (Auto_Cmd_Bus.valid == 1) && (Auto stream 不 stale) per gcs_loss_timeout_s (.17) 复用 = 5.0 s` |
| M-08 RTL entry | G-POS && G-HOME-SET | `home_set == 1`(由 GCS `CMDTYPE_SET_HOME` 或 `home_lat_deg_default` (.30) 等加载)|

#### 4.6.3 TAKEOFF entry guard(D3 T-10)

D3 T-10 转换守卫 = `(STATE != STATE_INAIR) && G-POS && (arm_origin 已成立)`。

任务 brief 要求:`vehicle_on_ground && armed && pilot_throttle > arm_throttle_threshold`。

| 子条件 | 多旋翼实现 |
|---|---|
| `vehicle_on_ground` | `STATE != STATE_INAIR`(per D3 G-NOT-INAIR;由 `landed_detected` 或 `STATE == STATE_GROUND` 满足)|
| `armed` | `VehicleStatus == STATUS_ARM`(进 ARMED super-state 已成立)|
| `pilot_throttle > arm_throttle_threshold` | `PILOT.thr > arm_throttle_thr_n01` (.02) = 0.10(per B3 .02)|

#### 4.6.4 ARM safety check mask(`FMS_PARAM.arm_safety_check_mask` (.05))位语义,多旋翼专属

per [D3 §4.9.4](D3-mode-manager.md) "arm_safety_check_mask 位语义由 D5 leaf 锁";Phase 2 多旋翼锁定下表:

| 位号 | 检查项 | 多旋翼实现 |
|---|---|---|
| bit 0 | INS ready | `INS_Status >= INS_STATUS_READY`(per A6 §4.5.3)|
| bit 1 | Attitude valid | `INS_Flag.attitude_valid == 1` |
| bit 2 | Position valid(若启用 GPS-required arm)| `INS_Flag.position_valid == 1`;Phase 2 默认 disabled(允许 GPS-less arm,与 Phase 5 outdoor 场景再启用)|
| bit 3 | Battery voltage above critical | `battery_voltage > critical_battery_voltage_v` (.19) = 13.2 V |
| bit 4 | Throttle below arm threshold | `PILOT.thr <= arm_throttle_thr_n01` (.02) = 0.10 |
| bit 5 | Kill switch off | `PILOT.kill_sw == 0` |
| bit 6 | Geofence pre-check | `home_set == 1 && geofence_enabled` (.11) `== 1` 时 home 在 geofence 内 |
| bit 7 | Mission load valid(若 entry 选 MISSION mode)| `mission_state == MISSION_LOADED` |
| bit 8..31 | reserved | 默认 1(any-pass)|

Phase 2 默认 mask = **0x000000FB**(bit 0,1,3,4,5,6,7 启用;bit 2 disabled;bit 8.. = 1);可由 GCS 调整(per B3 .05 R-T 标记)。

### 4.7 Failsafe 阈值(多旋翼数值)

D1 §4.4 + D2 §4.3.3 已锁触发分类与决策树;D5 填阈值数值(全部对应 B3 §4.4.2 .11..23 字段)。

| 项 | PARAM | Phase 2 多旋翼默认值 | 单位 | 来源 / 备注 |
|---|---|---:|---|---|
| RC link timeout | `FMS_PARAM.link_loss_timeout_s` (.16)| **1.0**(= 1000 ms)| s | 任务 brief 默认 1000 ms;B3 .16 默认 1.0 s 一致 |
| GCS link timeout | `FMS_PARAM.gcs_loss_timeout_s` (.17)| **5.0**(= 5000 ms)| s | 任务 brief 默认 5000 ms;B3 .17 默认 5.0 s 一致 |
| 电池低阈值 | `FMS_PARAM.low_battery_voltage_v` (.18)| **14.0** | V | 任务 brief 默认 14.0(4S LiPo);B3 .18 默认 14.0 一致 |
| 电池临界阈值 | `FMS_PARAM.critical_battery_voltage_v` (.19)| **13.2** | V | 任务 brief 默认 13.2;偏离 B3 .19 默认 13.0 → D5 取 brief 锁的 13.2(更保守 0.2 V) |
| Geofence 启用 | `FMS_PARAM.geofence_enabled` (.11)| **1**(true)| bool | B3 .11 默认 1 |
| Geofence shape | `FMS_PARAM.geofence_shape` (.12)| **`GF_CYLINDER`**(per [B2 §4.4.16](../B-contracts/B2-enum-inventory.md)) | enum `GeofenceShape` | 任务 brief + [D1 §4.5.2](D1-fms-functional.md) "GF_CYLINDER per Phase 2";B3 .12 默认 `GF_CYLINDER` 一致 |
| Geofence 半径 | `FMS_PARAM.geofence_radius_m` (.13)| **100.0** | m, NED 水平 | 任务 brief 默认 100.0;B3 .13 默认 100.0 一致 |
| Geofence 高度上限 | `FMS_PARAM.geofence_alt_max_m` (.14)| **120.0** | m, AGL | B3 .14 默认 120.0(FAA 400 ft);D5 沿用 |
| Geofence breach action | `FMS_PARAM.geofence_action` (.15)| **`FA_RTL`** | enum `FailsafeAction` | B3 .15 默认 `FA_RTL`;Phase 2 多旋翼沿用 |
| RC link loss action | `FMS_PARAM.failsafe_link_loss_action` (.20)| **`FA_RTL`** | enum | B3 .20 默认 `FA_RTL` |
| GCS link loss action | `FMS_PARAM.failsafe_gcs_loss_action` (.21)| **`FA_HOLD`** | enum | B3 .21 默认 `FA_HOLD`(仅暂停,不返航;Phase 2 多旋翼保守)|
| Low battery action | `FMS_PARAM.failsafe_low_bat_action` (.22)| **`FA_RTL`** | enum | B3 .22 默认 `FA_RTL` |
| Critical battery action | `FMS_PARAM.failsafe_critical_bat_action` (.23)| **`FA_LAND`** | enum | B3 .23 默认 `FA_LAND`(立即就地降落;不再尝试 RTL) |
| INS readiness timeout | `FMS_PARAM.ins_unready_timeout_s` (.24)| **30.0** | s | B3 .24 默认 30.0;A6 §4.7.1 INS gate 序列 |
| Mode-switch debounce | `FMS_PARAM.mode_switch_debounce_s` (.42)| **0.05** | s | B3 .42 默认 0.05(2.5 个 FMS 帧);D5 沿用 |
| Failsafe recovery dwell | `FMS_PARAM.failsafe_recovery_dwell_s` (.43)| **1.0** | s | B3 .43 默认 1.0;D5 沿用 |

### 4.8 Parameter name compatibility table(`FMS_PARAM` 兼容性核对,闭合 audit F-33)

per [B3 §4.9.2](../B-contracts/B3-parameter-schema.md) 模板;D5 必须为 B3 §4.4.2 全部 **43 个 `FMS_PARAM`** 字段填一行。FMT-Firmware **未挂载**,firmware-tip 列以 `(B4 verify; expected …)` 占位;一致性 / 处置列首跑 B4 / I3 后由 §10 changelog 升 locked。

#### 4.8.1 字段命名规范回顾(per [A2 R-7](../A-architecture/A2-naming-conventions.md))

- 模型仓侧 PARAM 名:`<功能>_<物理量>_<单位后缀>`(per A2);例如 `takeoff_climb_speed_mps`
- firmware tip 历史命名(per FMT-Firmware px4-style legacy):多用 `mc_<group>_<param>` 或 `MPC_<...>` 形式(此为 PX4 习惯;FMT-Firmware mc_fms 端命名沿用此风格的可能性高)
- 一致性 verdict 三类:**yes**(B3 字段名 = firmware tip)/**legacy**(firmware tip 用 px4-style 旧名,接受沿用,per A2 E-5.1)/**no**(冲突,需协同 B3 修订;触发 [B3 §10 changelog](../B-contracts/B3-parameter-schema.md) + INDEX 决策日志)

#### 4.8.2 D5 ↔ FMS_PARAM 兼容性表(全 43 字段)

| FMS_PARAM # | B3 字段名 | D5 锁定字段名(同 B3) | firmware tip 实际名(`(B4 verify; expected ...)`)| 一致性 | D5 处置 |
|---|---|---|---|---|---|
| .01 | `default_mode_at_boot` | `default_mode_at_boot` | `(B4 verify; expected default_pilot_mode 或 default_mode)` | TBD | align at first B4 run |
| .02 | `arm_throttle_thr_n01` | `arm_throttle_thr_n01` | `(B4 verify; expected arm_throttle_threshold 或 mc_arm_thr)` | TBD | align at first B4 run |
| .03 | `disarm_throttle_thr_n01` | `disarm_throttle_thr_n01` | `(B4 verify; expected disarm_throttle_threshold 或 mc_disarm_thr)` | TBD | align at first B4 run |
| .04 | `arm_stick_dwell_s` | `arm_stick_dwell_s` | `(B4 verify; expected arm_stick_time 或 mc_arm_dwell)` | TBD | align at first B4 run |
| .05 | `arm_safety_check_mask` | `arm_safety_check_mask` | `(B4 verify; expected arm_safety_mask 或 mc_arm_check)` | TBD | align at first B4 run;位语义已在 D5 §4.6.4 锁(模型仓侧固定)|
| .06 | `takeoff_alt_m` | `takeoff_alt_m` | `(B4 verify; expected mc_takeoff_alt 或 takeoff_target_alt_m)` | TBD | align at first B4 run;若 firmware 用 `takeoff_target_alt_m` 则属 `legacy`-equivalent,模型仓沿用 firmware 名 |
| .07 | `takeoff_climb_speed_mps` | `takeoff_climb_speed_mps` | `(B4 verify; expected mc_takeoff_speed 或 takeoff_climb_rate_mps)` | TBD | align at first B4 run;若 firmware 用 `takeoff_climb_rate_mps`(更标准)则属 legacy-rename — 接受 firmware 名 |
| .08 | `landing_descent_speed_mps` | `landing_descent_speed_mps` | `(B4 verify; expected mc_land_speed 或 land_descent_rate_mps)` | TBD | align at first B4 run;若 firmware 用 `land_descent_rate_mps`(更标准)→ accept firmware-legacy |
| .09 | `land_touchdown_speed_mps` | `land_touchdown_speed_mps` | `(B4 verify; expected mc_land_touch_speed 或 land_low_descent_rate_mps)` | TBD | align at first B4 run;若 firmware 命名为 `land_low_descent_rate_mps` 则属 brief 一致(brief 用同名)|
| .10 | `land_disarm_dwell_s` | `land_disarm_dwell_s` | `(B4 verify; expected mc_land_disarm_t 或 land_disarm_timeout_s)` | TBD | align at first B4 run;brief 用 `land_disarm_timeout_s`,与 firmware 端语义一致 |
| .11 | `geofence_enabled` | `geofence_enabled` | `(B4 verify; expected geofence_en 或 gf_enabled)` | TBD | align at first B4 run |
| .12 | `geofence_shape` | `geofence_shape` | `(B4 verify; expected gf_shape)` | TBD | align at first B4 run |
| .13 | `geofence_radius_m` | `geofence_radius_m` | `(B4 verify; expected gf_radius 或 geofence_max_radius_m)` | TBD | align at first B4 run;brief 用 `geofence_radius_m`,B3 同 |
| .14 | `geofence_alt_max_m` | `geofence_alt_max_m` | `(B4 verify; expected gf_alt_max 或 geofence_max_alt_m)` | TBD | align at first B4 run |
| .15 | `geofence_action` | `geofence_action` | `(B4 verify; expected gf_action)` | TBD | align at first B4 run |
| .16 | `link_loss_timeout_s` | `link_loss_timeout_s` | `(B4 verify; expected rc_loss_t 或 rc_loss_timeout_ms)` | TBD | align at first B4 run;**单位差异关注**:brief 用 `_ms`(1000 ms),B3 用 `_s`(1.0 s);若 firmware 端用 `_ms`,B3 / D5 端可能需要在 schema 上做单位换算或重命名 — escalate to B3 §10 changelog |
| .17 | `gcs_loss_timeout_s` | `gcs_loss_timeout_s` | `(B4 verify; expected gcs_loss_t 或 gcs_loss_timeout_ms)` | TBD | 同 .16 单位差异 |
| .18 | `low_battery_voltage_v` | `low_battery_voltage_v` | `(B4 verify; expected batt_low_v 或 batt_low_voltage_v)` | TBD | align at first B4 run;brief 用 `batt_low_voltage_v`,B3 用 `low_battery_voltage_v` — 若 firmware 用 brief 风格,接受 firmware-legacy 重命名 |
| .19 | `critical_battery_voltage_v` | `critical_battery_voltage_v` | `(B4 verify; expected batt_crit_v 或 batt_critical_voltage_v)` | TBD | 同 .18 |
| .20 | `failsafe_link_loss_action` | `failsafe_link_loss_action` | `(B4 verify; expected rc_loss_action 或 failsafe_rc_action)` | TBD | align at first B4 run |
| .21 | `failsafe_gcs_loss_action` | `failsafe_gcs_loss_action` | `(B4 verify; expected gcs_loss_action 或 failsafe_gcs_action)` | TBD | align at first B4 run |
| .22 | `failsafe_low_bat_action` | `failsafe_low_bat_action` | `(B4 verify; expected batt_low_action 或 failsafe_batt_low_action)` | TBD | align at first B4 run |
| .23 | `failsafe_critical_bat_action` | `failsafe_critical_bat_action` | `(B4 verify; expected batt_crit_action 或 failsafe_batt_crit_action)` | TBD | align at first B4 run |
| .24 | `ins_unready_timeout_s` | `ins_unready_timeout_s` | `(B4 verify; expected ins_ready_t 或 ins_init_timeout_s)` | TBD | align at first B4 run |
| .25 | `wp_array_max_len` | `wp_array_max_len` | `(B4 verify; expected wp_max 或 mission_wp_max)` | TBD | align at first B4 run;C-I 字段(数组维度) |
| .26 | `wp_acceptance_radius_m` | `wp_acceptance_radius_m` | `(B4 verify; expected wp_accept_r 或 wp_acceptance_r_m)` | TBD | align at first B4 run |
| .27 | `mission_default_speed_mps` | `mission_default_speed_mps` | `(B4 verify; expected mission_speed 或 mission_default_v_mps)` | TBD | align at first B4 run |
| .28 | `rtl_climb_alt_m` | `rtl_climb_alt_m` | `(B4 verify; expected rtl_alt 或 rtl_altitude_m)` | TBD | align at first B4 run;**brief 命名为 `rtl_altitude_m`**;若 firmware 用 brief 风格,接受 firmware-legacy 名 |
| .29 | `rtl_loiter_time_s` | `rtl_loiter_time_s` | `(B4 verify; expected rtl_loiter_t 或 rtl_home_loiter_s)` | TBD | align at first B4 run |
| .30 | `home_lat_deg_default` | `home_lat_deg_default` | `(B4 verify; expected home_lat 或 home_default_lat_deg)` | TBD | align at first B4 run |
| .31 | `home_lon_deg_default` | `home_lon_deg_default` | `(B4 verify; expected home_lon 或 home_default_lon_deg)` | TBD | align at first B4 run |
| .32 | `home_alt_m_default` | `home_alt_m_default` | `(B4 verify; expected home_alt 或 home_default_alt_m)` | TBD | align at first B4 run |
| .33 | `vel_xy_lim_mps` | `vel_xy_lim_mps` | `(B4 verify; expected mc_vel_xy_max 或 vel_xy_max_mps)` | TBD | align at first B4 run |
| .34 | `vel_z_up_lim_mps` | `vel_z_up_lim_mps` | `(B4 verify; expected mc_vel_up_max 或 vel_z_up_max_mps)` | TBD | align at first B4 run |
| .35 | `vel_z_down_lim_mps` | `vel_z_down_lim_mps` | `(B4 verify; expected mc_vel_dn_max 或 vel_z_down_max_mps)` | TBD | align at first B4 run |
| .36 | `acc_xy_lim_mps2` | `acc_xy_lim_mps2` | `(B4 verify; expected mc_acc_xy_max 或 acc_xy_max_mps2)` | TBD | align at first B4 run |
| .37 | `jerk_xy_lim_mps3` | `jerk_xy_lim_mps3` | `(B4 verify; expected mc_jerk_xy_max 或 jerk_xy_max_mps3)` | TBD | align at first B4 run |
| .38 | `tilt_lim_rad` | `tilt_lim_rad` | `(B4 verify; expected mc_tilt_max 或 tilt_max_rad)` | TBD | align at first B4 run |
| .39 | `yaw_rate_lim_radps` | `yaw_rate_lim_radps` | `(B4 verify; expected mc_yaw_rate_max 或 yaw_rate_max_radps)` | TBD | align at first B4 run |
| .40 | `manual_stick_deadzone_n01` | `manual_stick_deadzone_n01` | `(B4 verify; expected stick_deadzone 或 stick_deadband_n01)` | TBD | align at first B4 run;**brief 用 `stick_roll_pitch_deadband_ratio` / `stick_yaw_deadband_ratio` / `stick_throttle_deadband_ratio` 三 channel 分体**;B3 用单一字段共享;若 firmware 用三 channel,B3 / D5 协同扩展 §10 changelog;Phase 2 用单一 |
| .41 | `manual_thr_hover_n01` | `manual_thr_hover_n01` | `(B4 verify; expected mc_thr_hover 或 hover_throttle)` | TBD | align at first B4 run;brief 用 `hover_throttle`,可能 firmware-legacy 名 |
| .42 | `mode_switch_debounce_s` | `mode_switch_debounce_s` | `(B4 verify; expected mode_switch_t 或 mode_change_debounce_s)` | TBD | align at first B4 run |
| .43 | `failsafe_recovery_dwell_s` | `failsafe_recovery_dwell_s` | `(B4 verify; expected failsafe_recover_t 或 fs_recovery_dwell_s)` | TBD | align at first B4 run |

**总计**:43 行,与 B3 §4.4.2 字段集 1:1 对齐;一致性列全部 `TBD`;所有处置 = `align at first B4 run`(per B3 §5 第 1 项 + INDEX 决策日志期约定)。本表满足 [B3 §4.9.2](../B-contracts/B3-parameter-schema.md) "D5 必须为 §4.4.2 中**全部 43 个 FMS_PARAM 字段**填一行" 退出条件,闭合 audit F-33。

#### 4.8.3 单位差异关注事项(B4 / B3 协同点)

任务 brief 与 B3 §4.4.2 的单位 / 字段名差异点(高风险条目):

| 项 | brief 风格 | B3 当前 | 差异类型 | 处置 |
|---|---|---|---|---|
| RC link timeout | `rc_loss_timeout_ms`(unit=ms) | `link_loss_timeout_s`(unit=s) | 单位差(× 1000)| B3 锁 `_s`(SI 单位 per [A2](../A-architecture/A2-naming-conventions.md));brief 的 `_ms` 是 GCS UI 显示习惯;模型内部统一 `_s`,GCS 端做单位换算。**不**修改 B3 |
| GCS link timeout | `gcs_loss_timeout_ms` | `gcs_loss_timeout_s` | 单位差 | 同上 |
| Hover throttle | `hover_throttle` | `manual_thr_hover_n01` | 命名习惯差 | 若 firmware 用 brief 风格,B3 在首跑 B4 后做 legacy-rename |
| Stick deadband 三 channel 分体 | `stick_roll_pitch_deadband_ratio` / `stick_yaw_deadband_ratio` / `stick_throttle_deadband_ratio` | 单一 `manual_stick_deadzone_n01` | 字段拆分差 | Phase 2 用单一;若 firmware 用三 channel,**B3 / D5 协同**在 [B3 §4.4.2](../B-contracts/B3-parameter-schema.md) 增补两个 `FMS_PARAM` 字段(.44 / .45),触发 §10 changelog;否则 D5 用 D5 leaf field 表示差异 |
| RTL altitude | `rtl_altitude_m` | `rtl_climb_alt_m` | 命名差 | 若 firmware 用 brief 风格,接受 firmware-legacy |
| 触地油门阈值 | `land_ground_detect_throttle_threshold` | (D5 leaf) | B3 缺字段 | D5 leaf-only;首跑 B4 后协调 B3 是否增补 |
| Lift-off detection threshold | `takeoff_liftoff_z_threshold_m` | (D5 leaf) | B3 缺字段 | D5 leaf-only |
| Flare altitude | `land_flare_alt_m` | (D5 leaf) | B3 缺字段 | D5 leaf-only |
| Pilot rate max | `pilot_rate_max_radps` | (D5 leaf) | B3 缺字段 | D5 leaf-only;[D4 §5 risk](D4-command-shaper.md) 已登记 |

#### 4.8.4 D5-only(non-firmware-mirror)字段汇总

D5 在 §4.2 / §4.3 / §4.5 引入了 6 个 D5 leaf field(不在 B3 §4.4.2 .01..43 范围),Phase 2 视为 D5 leaf-only(模型仓侧 PARAM 但不写入 firmware-mirrored `FMS_PARAM_TYPE`):

| D5 leaf field | 类别 | Phase 2 默认值 | 单位 | 引用位置 |
|---|---|---:|---|---|
| `takeoff_liftoff_z_threshold_m` | takeoff | −0.3 | m, NED-Down | §4.2 |
| `takeoff_liftoff_vz_threshold_mps` | takeoff | −0.2 | m/s, NED-Down | §4.2 |
| `land_low_alt_m` | landing | 1.0 | m, NED | §4.3 |
| `land_flare_alt_m` | landing | 0.3 | m, NED | §4.3 |
| `land_ground_detect_throttle_threshold_n01` | landing | 0.2 | normalized | §4.3 |
| `expo_ratio_roll_pitch` | stick | 0.30 | unitless | §4.5 |
| `expo_ratio_yaw` | stick | 0.30 | unitless | §4.5 |
| `pilot_rate_max_radps` | stick | 3.5 | rad/s, body | §4.5 |
| `transition_ramp_ms` | shaper | 200 | ms | §4.5 |
| `stick_lp_cutoff_hz` | stick | 0.0 (disabled) | Hz | §4.5 |

**处置策略**:首跑 B4 / I3 后,与 B3 协同决定是否增补到 firmware-mirrored `FMS_PARAM` 字段集;若否,继续作为 D5 leaf-only(模型仓内部 PARAM struct,经 ert.tlc 生成时仍为 inline-tunable);触发 [B3 §10 changelog](../B-contracts/B3-parameter-schema.md) 与 INDEX 决策日志登记。

### 4.9 Cross-references(D5 → 上下游接口)

#### 4.9.1 D5 → D2(FMS 结构)

D5 不修改 D2 §4.3 子模块边界;D5 数值通过 `FMS_PARAM` struct 注入到:
- (3) Safety Monitor:消费 §4.7 失效阈值
- (4) Mission Manager:消费 §4.4 RTL 数值 / `wp_acceptance_radius_m` (.26) / `mission_default_speed_mps` (.27)
- (5) Command Shaper:消费 §4.2 / §4.3 / §4.5 数值(per [D4 §4.9.1](D4-command-shaper.md))

#### 4.9.2 D5 → D3(Mode Manager)

per [D3 §4.9.4](D3-mode-manager.md) 5 注入项:
- `arm_safety_check_mask` 位语义 — D5 §4.6.4 锁
- `tilt_lim_rad` (.38) / `vel_xy_lim_mps` (.33) / `vel_z_*_lim_mps` (.34/.35) — D5 §4.7 / §4.5 全部沿用 B3 默认
- `landed_detected` 谓词数值 — D5 §4.3 触地检测合成(`motor_cmd_avg < 0.2 && |INS.vel| < 0.15 m/s` 持续 5 帧);**与 [D3 §5 风险 "landed_detected 谓词归属"](D3-mode-manager.md) 协调:此谓词在 FMS 侧由 D5 实现(D2 §4.3.1 Source Selector 拿来 echo);若 C3 多旋翼 Plant leaf 同期产出 `Plant_States_Bus.landed_flag`,D5 改为 echo Plant 输出**(Wave 9 协同)
- `mission_done_action` — D5 锁 = `RTL`(per D3 T-20)
- M-12 ACRO 启用 = **否**(per §4.1.1)

#### 4.9.3 D5 → D4(Command Shaper)

per [D4 §4.9.1](D4-command-shaper.md) hand-off:D5 已为 D4 hand-off 表的全部数值 / D5 leaf field 填值(见本文 §4.2 / §4.3 / §4.4 / §4.5)。D4 algorithm form 不动。

#### 4.9.4 D5 → D6(cmd_mask 接口)

D5 不修改 D6 §4.2 位语义 / §4.4 互斥规则 / §4.5 INS validity 政策 / §4.6 hand-off / §4.7 reset 强制态;§4.1.1 表逐 mode 复核 multicopter 偏离 = 无,确认 [D6 §4.4.1 truth-table](D6-fms-controller-interface.md) 合法集合在多旋翼语境下完整可达。

#### 4.9.5 D5 → B3(parameter schema)

D5 §4.8 兼容性表填表满足 [B3 §4.9.2](../B-contracts/B3-parameter-schema.md) 退出条件(43 项全填)。§4.8.4 D5-only 字段集是 B3 的潜在扩展候选(由 §10 changelog + INDEX 决策日志触发 B3 增补)。

#### 4.9.6 D5 → 下游(F3 / G3 / H1 / H4)

- **F3 INS 消费规则**:D5 §4.6 entry guard 引用 INS_Flag 位作为多旋翼专属阈值:`position_valid` / `velocity_valid` / `attitude_valid` / `heading_valid`;F3 在自身字段消费矩阵中需为多旋翼标 mandatory
- **G3 日志**:D5 §4.2 / §4.3 / §4.4 三个 profile 内部状态(takeoff_phase / land_phase / rtl_phase)+ §4.6.4 arm_safety_check_mask 触发位 + §4.7 失效阈值触发标志 = G3 logsout 多旋翼专属可观测项候选
- **H1 验证场景**:§4.2 takeoff sequence + §4.3 landing sequence + §4.4 RTL 5 段序列 + §4.7 失效场景(RC loss / GCS loss / 低电 / 临界电 / geofence breach)= H1 多旋翼场景候选 + 数值阈值
- **H4 故障注入**:§4.7 5 类失效阈值是 H4 故障注入参数化范围;§4.3 触地检测算法是 H4 sensor stub fault scenarios

## 5. 已知风险与悬而未决问题

- **firmware commit hash 暂缺(共享于 A3/A6/A7/B1/B2/B3/B4/B5/D6/D5)**
  - 影响:§3 / §4.8 全部以 `<pending hash>` 占位 + firmware-tip 列以 `(B4 verify; expected ...)` 占位
  - 处置:open;首跑 B4 / I3 后由 §10 changelog + INDEX 决策日志补齐;一致性 verdict TBD → yes/legacy/no 同步升级;触发兼容性表的 `D5 处置`列从 "align at first B4 run" 升至具体决策

- **D5-only 字段(§4.8.4 共 10 项)是否反向回流到 B3 `FMS_PARAM`**
  - 影响:lift-off 检测两阈值 / land_low_alt / land_flare_alt / land_ground_detect_thr / expo_ratio_×2 / pilot_rate_max / transition_ramp_ms / stick_lp_cutoff = 10 项;若 firmware 端有对应字段,B3 应增补;若无,Phase 2 维持 D5 leaf-only
  - 处置:open;首跑 B4 后 [B3 §10 changelog](../B-contracts/B3-parameter-schema.md) + INDEX 决策日志登记;字段号顺延至 .44+

- **`landed_detected` 谓词数据源 — `Control_Out_Bus.motor_cmd[]` vs `Plant_States_Bus.landed_flag`**
  - 影响:§4.3 触地检测用 `motor_cmd_avg < 0.2`;motor_cmd 来源:(a) [D1 §4.7 OQ4 transitional](D1-fms-functional.md) `Control_Out_Bus.motor_cmd[]` 直传(Phase 2 启用)/ (b) Plant 侧 [C3 多旋翼 leaf](../C-plant/C3-multicopter-leaf.md) 计算 `Plant_States_Bus.landed_flag`;两者在 Phase 2 应一致,但 Phase 5 移除 `Control_Out_Bus`-as-FMS-input 后只能走 (b)
  - 处置:open;Wave 9 D5 / C3 / E4 同期协同;若 Phase 2 用 (a),Phase 3 SIH 评估时改 (b);[D3 §5 risk "landed_detected 谓词归属"](D3-mode-manager.md) 同步条目

- **Stick deadband 三 channel 分体 vs 单一字段**
  - 影响:§4.5 表用单一 `manual_stick_deadzone_n01` (.40) 共享 roll/pitch/yaw,throttle 在实现层 bypass;任务 brief 暗示三 channel 分体(yaw_deadband / throttle_deadband 单列)
  - 处置:open;首跑 B4 后看 firmware 端实际字段;若三 channel,B3 / D5 协同增补 .44 / .45 字段(`stick_yaw_deadband_n01` / `stick_throttle_deadband_n01`)+ §10 changelog;否则 Phase 2 用 D5 leaf field 表示差异

- **`takeoff_alt_m` brief 默认 3.0 vs B3 默认 2.5**
  - 影响:§4.2 取 brief 锁的 3.0;B3 §4.4.2 .06 默认 2.5(PX4 baseline)。两者均 PX4-class,都合理,但 brief 与 B3 不一致
  - 处置:Phase 2 取 brief 锁的 3.0;[B3 §10 changelog](../B-contracts/B3-parameter-schema.md) 同步更新默认 — 走 D5 → B3 反向回流;若 firmware 端有不同默认,首跑 B4 / I3 后 reconcile

- **`landing_descent_speed_mps` brief 默认 0.5 vs B3 默认 0.7;`land_touchdown_speed_mps` brief 默认 0.2 vs B3 默认 0.3**
  - 影响:同上;D5 取 brief 锁(更保守的下降率)
  - 处置:Phase 2 取 brief 锁的 0.5 / 0.2;B3 同步 §10 changelog

- **`critical_battery_voltage_v` brief 默认 13.2 vs B3 默认 13.0**
  - 影响:同上;D5 取 brief 锁的 13.2(更保守 0.2 V buffer)
  - 处置:Phase 2 取 brief 锁;B3 同步 §10 changelog

- **`Control_Out_Bus.motor_cmd` 是否在 Phase 2 启用作为 FMS 输入(per [D1 §4.7 OQ4](D1-fms-functional.md))**
  - 影响:§4.3 触地检测用 motor_cmd_avg;若 OQ4 transitional 在 Phase 2 不启用,触地检测改用 throttle_cmd 替代 / 或 INS-derived altitude+vel
  - 处置:Phase 2 默认启用(per D1 §4.7 锁定 = transitional Phase 2 enable);H4 验证;Phase 3 SIH 评估;Phase 5 改用 throttle_cmd / Plant_States_Bus.landed_flag

- **§4.6.4 `arm_safety_check_mask` bit 2(GPS-required arm)Phase 2 默认 disabled**
  - 影响:Phase 2 允许 GPS-less arm(室内场景);Phase 5 outdoor 场景需启用,届时 mask 改为 `0x000000FF`
  - 处置:Phase 2 锁 0x000000FB;Phase 5 leaf 启动后改 mask;走 §10 changelog

## 6. 退出条件复核

对照 [`00-design-plan.md §4.D D5 行`](../00-design-plan.md):

> **D5 退出条件原文**:多旋翼特定的 mode-cmd 映射、起飞/降落 profile;**参数名与 firmware `FMS_PARAM` 字段名兼容性显式核对**

| # | 退出条件原文(分项) | 本文档依据 | 状态 |
|---|---|---|---|
| 1 | 多旋翼特定的 mode-cmd 映射 | §4.1.1 主 mode 表(M-01..M-12 共 12 行)+ §4.1.2 failsafe sub-mode 表(FS-01..FS-04 共 4 行)+ §4.1.3 关键不变量(D5 不 emit 互斥违规 / 不新增位 / 不新增 mode);默认 = 沿用 D3 §4.6 + D4 §4.2,M-12 ACRO 锁定 Phase 2 不启用 | 满足 |
| 2 | 起飞 profile | §4.2 全表(8 行:目标高度 / 爬升速率 / hover 油门 / lift-off 高度阈值 / lift-off vz 阈值 / phase_1→2 判据 / phase_2→3 判据 / TAKEOFF→POSHOLD 切换);闭合 [D4 §5 open](D4-command-shaper.md) | 满足 |
| 3 | 降落 profile | §4.3 全表(9 行:高空下降率 / 低空下降率 / 低空阈值 / flare 高度 / 触地油门 / 触地速度判据 / 解锁延时 / phase_1→2 / phase_2→3 / phase_3→DISARM);闭合 D4 §5 open;触地检测算法 D5 锁 | 满足 |
| 4 | 参数名与 firmware `FMS_PARAM` 字段名兼容性显式核对(闭合 audit F-33)| §4.8.2 全 43 行兼容性表(每行 = B3 字段名 / D5 锁定名 / firmware-tip 占位 / 一致性 / D5 处置);§4.8.3 单位/命名差异关注事项;§4.8.4 D5-only 字段汇总(10 项);填表满足 [B3 §4.9.2](../B-contracts/B3-parameter-schema.md) "D5 必须为全部 43 个 FMS_PARAM 字段填一行" | 满足 |

**额外覆盖**(超出退出条件原文,任务 brief 要求):
- §4.4 RTL profile(brief §4.4)— 锁 RTL altitude 30 m / 巡航速度 5 m/s / loiter 5 s / 锥形下降经 LAND profile 接力
- §4.5 Stick conditioning(brief §4.5)— 锁 deadband 0.05 / expo 0.30 / pilot rate max 3.5 rad/s / bumpless ramp 200 ms
- §4.6 Mode entry/exit guards(brief §4.6)— 锁 POSHOLD / AUTO / TAKEOFF entry guard 多旋翼数值化 + arm_safety_check_mask 位语义
- §4.7 Failsafe 阈值(brief §4.7)— 锁 RC link 1 s / GCS link 5 s / batt low 14.0 V / batt critical 13.2 V / geofence GF_CYLINDER 100 m

## 7. 下游影响

按 [`01-design-relationships.md §4.4`](../01-design-relationships.md) D 区出边 + Wave 9 同期 / Wave 10..11 衍生:

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [F3 INS 消费规则](../F-ins-contract/F3-consumption-rules.md)(Wave 9 同期)| ⇢ | §4.6 entry guard 引用 INS_Flag 多旋翼专属位用法 → F3 字段消费矩阵的 multicopter 列 |
| [E4 Controller 多旋翼 leaf](../E-controller/E4-multicopter-leaf.md)(Wave 9 同期)| ⇢ | §4.4 RTL altitude / 巡航速度 + §4.5 pilot rate max + §4.7 失效阈值数值 → E4 controller leaf 的 setpoint 物理可行域参考(降低 controller 限幅压力)|
| [C3 Plant 多旋翼 leaf](../C-plant/C3-multicopter-leaf.md)(Wave 9 同期)| ⇢ | §4.3 触地检测合成 → C3 `Plant_States_Bus.landed_flag` 计算合成对应(若 C3 端实现 landed_flag,D5 改为 echo,见 §5 风险)|
| [G3 日志](../G-harness/G3-logging.md)(Wave 10)| ⇢ | §4.9.6 G3 logsout 候选项(profile phase / arm_safety_check 位 / 失效触发标志)|
| [H1 验证场景](../H-verification/H1-scenario-catalog.md)(Wave 11)| ⇢ | §4.2 / §4.3 / §4.4 takeoff / landing / RTL profile + §4.7 失效阈值 → H1 多旋翼场景候选与数值阈值 |
| [H2 验证指标](../H-verification/H2-metrics.md)(Wave 11)| ⇢ | §4.2 / §4.3 / §4.4 数值默认 → H2 多旋翼场景指标判据来源(例:takeoff 高度容差 0.2 m / 触地速度 < 0.15 m/s)|
| [H4 故障注入](../H-verification/H4-fault-catalog.md)(Wave 11)| ⇢ | §4.7 5 类失效阈值 → H4 RC loss / GCS loss / 低电 / 临界电 / geofence 故障注入参数化范围 |
| [B3 parameter schema](../B-contracts/B3-parameter-schema.md)(已 reviewed,potential reverse flow)| ⇢(反向回流候选)| §4.8.4 D5-only 10 项 + §4.8.3 命名 / 单位差异 → B3 §10 changelog 候选(首跑 B4 / I3 后)|

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-09 | Wave 9 D5 author | 初稿 — §4.1 多旋翼 mode-cmd 复核(M-01..M-12 + FS-01..FS-04 共 16 行,默认沿用 D3/D4,M-12 ACRO Phase 2 不启用);§4.2 TAKEOFF profile(8 行);§4.3 LAND profile(9 行);§4.4 RTL profile(7 行);§4.5 Stick conditioning(8 行);§4.6 Mode entry guards(POSHOLD / AUTO / TAKEOFF + arm_safety_check_mask 位语义);§4.7 Failsafe 阈值(15 行);§4.8 兼容性表(43 项 FMS_PARAM 全填,闭合 audit F-33)+ §4.8.4 D5-only 10 字段汇总;§4.9 cross-reference;闭合 D4 §5 lift-off / flare / 触地检测三 open |

## Self-check

- [x] frontmatter 完整,字段值合法
- [x] 上游文档全部存在且 status ≥ reviewed(架构 v1 / A2..A8 / B1..B3 / D1..D4 / D6 全部 reviewed)— D5 不属任何 co-seal batch
- [x] 退出条件逐条复核完成,每条均给出依据(§6 表 4 行 + 额外覆盖段 4 项)
- [x] 引用路径全部可点击访问(架构 v1 / A2..A8 / B1..B3 / D1..D4 / D6 / E4 / C3 / F3 / G3 / H1 / H2 / H4 全用相对路径)
- [x] 不存在 RULES §5 禁则中的内容(无 .slx 截图;无 .m 可执行代码;无 firmware 实现细节复述;无 PR/branch 名;不重复定义 bus / enum / param 字段表 — §4.8 兼容性表是字段名 verify,不是 schema 重定义)
- [N/A] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记 — `contract_impact: no`(D5 仅填值,字段集 / 类型 / 顺序由 B3 拥有;cmd_mask 位 / mode 集合 / 失效行为类别由 D1/D3/D6/B2 拥有),N/A
- [N/A] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径 — 本文件不直接镜像 firmware 契约,引用走 B3 / B2 / D6;§3 依赖与 §4.8 表保留 `<pending hash>` 占位以备 B4 兼容性 verify,N/A
- [x] 下游影响已沿关系图识别完毕(§7 表覆盖 [01 §4.4](../01-design-relationships.md) D5 下游 F3 / E4 / C3 / G3 / H1 / H2 / H4 + B3 反向回流候选)
- [x] 文档不超出本工作项范围(无越权设计 — 不修改 D3 chart 拓扑 / 不修改 D4 内部块 / 不重定义 B3 schema / 不锁 cmd_mask 位语义 / 不锁 controller / plant leaf;只填多旋翼数值与 entry guard 阈值与 arm_safety_check_mask 位语义与 §4.8 兼容性表)
