---
work_item: D2
title: FMS 结构设计
upstream: [架构v1, A1, A2, A3, A4, A5, A6, A7, A8, B1, B2, B3, D1]
contract_impact: no
status: reviewed
authored_at: 2026-05-08
last_reviewed_at: 2026-05-08
reviewer_verdict: pass
---

# D2 FMS 结构设计

## 1. 目的

按 [架构 v1 §12.1](../../architecture/2026-05-05-fmt-model-architecture-v1.md) 给定的 6 子模块**宏观拓扑**(Source Selector / Mode Manager / Safety Monitor / Mission Manager / Command Shaper / Output Assembler),把 [D1 FMS 功能设计](D1-fms-functional.md) 已锁定的 mode 集 / 命令源 / 整形策略 / 安全触发 / mission 范围 / 输出 bus 字段类别**映射到 6 个子模块的边界与内部信号流**,作为 [D3](D3-mode-manager.md) / [D4](D4-command-shaper.md) / [D5](D5-multicopter-leaf.md) / [D6](D6-fms-controller-interface.md) 的结构启动输入。

## 2. 范围

**在范围:**

- 6 个子模块的边界(输入 / 输出 / Reset 行为 / 速率 / Owner shared vs leaf)
- 子模块之间的内部信号流(intra-FMS bus 命名 + 字段类别,**不**字段级)
- D2 → D3(Mode Manager Stateflow)、D2 → D4(Command Shaper)、D2 → D6(cmd_mask 编排接口)、D2 → D5(多旋翼 leaf)的 hand-off 内容
- 变体策略下 D2 macro 与 D5 leaf 的接口规则(per [A5](../A-architecture/A5-variant-strategy.md))
- Init / Reset 的子模块协议(per [A6](../A-architecture/A6-init-reset-contract.md))

**不在范围(由其他工作项处理):**

- Mode Manager Stateflow 内部状态 / 转换 / 事件 / 守卫 — 由 [D3 Mode Manager 详细设计](D3-mode-manager.md) 处理(Wave 8 co-seal)
- Command Shaper 整形公式(rate/jerk-limit 数学、起飞/降落 profile 数值曲线) — 由 [D4 Command Shaper 详细设计](D4-command-shaper.md) 处理(Wave 8 co-seal)
- 多旋翼特定 mode-cmd 映射、起飞/降落 profile 数值参数、leaf 默认值 — 由 [D5 多旋翼 leaf](D5-multicopter-leaf.md) 处理(Wave 9)
- `cmd_mask` 每位**语义**(置位时 Controller 做什么) — 由 [D6 FMS↔Controller 接口约定](D6-fms-controller-interface.md) ↔ [E1 Controller 功能设计](../E-controller/E1-controller-functional.md) co-seal Wave 8 处理
- bus / enum / parameter 字段级 schema — 由 [B1](../B-contracts/B1-bus-inventory.md) / [B2](../B-contracts/B2-enum-inventory.md) / [B3](../B-contracts/B3-parameter-schema.md) 拥有
- INS_Out_Bus 字段消费的 fallback / validity 矩阵 — 由 [F3 INS 消费规则](../F-ins-contract/F3-consumption-rules.md) 处理(Wave 9)
- Harness 顶层、调度连接、跨速率 wiring — 由 [G1](../G-harness/G1-mil-toplevel.md) / [G2](../G-harness/G2-rate-scheduling.md) 处理

## 3. 依赖

| 上游 | 引用位置 | 用途 |
|---|---|---|
| 架构 v1 §7.1 / §8.3 / §12.1 / §13 / §10.4 | [架构 v1](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | FMS 模块定义、boundary、6 子模块官方拓扑(§12.1.1..§12.1.6)、20 ms 单速率约束(§13)、内部 bus 命名前缀约定(§10.4) |
| A1 v1 评审 — §4.5.3 / §17 OQ2 | [A1](../A-architecture/A1-v1-review.md) | A1 把 OQ2(保留 vs 重写)裁决指派给 D1;D1 已选 "macro 保留 + leaf 重写" → D2 必须沿用 6 子模块拓扑 |
| A2 命名与单位约定 | [A2](../A-architecture/A2-naming-conventions.md) | 内部 bus 命名(`FMS_*_Bus`)+ 子模块块名命名规则 |
| A3 模块边界 — §4.4(FMS 输入)/ §4.4.6(`FMS_Out_Bus` 输出)/ §4.6(passthrough 字段集 — passthrough-with-shape vs passthrough-direct 双类) | [A3](../A-architecture/A3-module-boundaries.md) | FMS 边界 bus 与字段;passthrough 双类决定 D2 中各 passthrough 字段的路由(经 D4 vs 直入 Output Assembler) |
| A4 跨速率边界 — RB-04 / RB-05 / RB-06 | [A4](../A-architecture/A4-rate-boundaries.md) | FMS 内部全部 20 ms;输入端走 RB-04(INS)/ RB-06(外部),输出端走 RB-05(到 Controller);D2 子模块均消费 latch 后副本 |
| A5 变体策略 | [A5](../A-architecture/A5-variant-strategy.md) | Phase 2 唯一 leaf = 多旋翼;6 子模块均归 shared,leaf 仅在 D5 锚 PARAM / profile 常量 |
| A6 init/reset 契约 — §4.4.2 `FMS_Out_Bus` reset 类别表 / §4.5.2 global vs local reset / §4.5.3 INS readiness gate | [A6](../A-architecture/A6-init-reset-contract.md) | 6 子模块 init / reset 行为、`FMS_init` 副作用边界、reset 后输出稳态值类别 |
| A7 时间约定 | [A7](../A-architecture/A7-time-conventions.md) | step interface timestamp 单源;子模块共用一个 timestamp,不各自打戳 |
| A8 共享库块 — §4.3.2 `model/fms/shared/` 7 块 | [A8](../A-architecture/A8-shared-library-roster.md) | 6 子模块的**框架壳**已在 A8 §4.3.2 列出(`FMS_Source_Selector_Shell` / `FMS_Mode_Manager_Framework` / `FMS_Safety_Monitor_Framework` / `FMS_Mission_Decoder_Shell` / `FMS_Command_Shaper_Shell` / `FMS_Output_Assembler_Shell` + `FMS_Reset_Hooks_Shell`);D2 每个子模块对应一个 A8 shell |
| B1 Bus 清单 — §4.2(FMS 输入)/ §4.7(`FMS_Out_Bus` ledger) | [B1](../B-contracts/B1-bus-inventory.md) | 边界 bus 字段级 schema(D2 引用) |
| B2 Enum 清单 — §4.4.1..14 | [B2](../B-contracts/B2-enum-inventory.md) | mode / state / ctrl_mode / failsafe_state / mission_state / cmd_mask 工作位号 |
| B3 Parameter schema | [B3](../B-contracts/B3-parameter-schema.md) | `FMS_PARAM` 字段(D2 仅引用 PARAM 名,数值由 B3 / D5 锁) |
| D1 FMS 功能设计(reviewed 2026-05-08) | [D1](D1-fms-functional.md) | mode 集合 / 命令源 / 整形策略 / 安全触发 / mission 范围 / 输出字段类别 / OQ2 macro-leaf 裁决(§4.8 → D2 必须采用 6 子模块拓扑)/ OQ4 transitional(`Control_Out_Bus` 作为 Safety + diagnostic 输入)|
| FMT-Firmware @ `<pending hash>` | `FMT-Firmware/src/model/fms/<vehicle>/lib/FMS_types.h` | 契约镜像源(D1 §4.8 已声明 leaf 重写 / macro 保留;D2 不直读 firmware,以 B1/B2/B3 已锁字段为准) |

环境注:FMT-Firmware 在本设计阶段**未挂载**;本文件以"D1 已锁功能 + A3 / A6 / A8 / B1 边界"派生子模块拓扑,不依赖 firmware 直接读取。

## 4. 设计内容

### 4.1 顶层 FMS 拓扑

#### 4.1.1 6 子模块顶层框图

per [架构 v1 §12.1.1..§12.1.6](../../architecture/2026-05-05-fmt-model-architecture-v1.md);所有内部信号同 20 ms 周期(per [A4 §4.4 RB-04/05/06](../A-architecture/A4-rate-boundaries.md) + 架构 v1 §13)。**框图箭头 = 内部 bus**(命名 §4.2);**虚线箭头** = 控制 / 请求(非 setpoint 数据)。

```text
                                 +---------- external buses (latched at FMS step start, RB-06/04) ----------+
                                 |                                                                          |
   Pilot_Cmd_Bus -----+          |    INS_Out_Bus (20ms ZOH from RB-04)                                     |
   GCS_Cmd_Bus -------+          |        |                                                                 |
   Auto_Cmd_Bus ------+          |        +--------+-----------+----------+                                 |
   Mission_Data_Bus -+|          |                 |           |          |                                 |
   Control_Out_Bus -+||  (D1 §4.7 transitional;诊断/cross-check only)                                       |
                   ||||          |                 |           |          |                                 |
                   vvvv          |                 v           v          v                                 |
                  +-----------+  |             +----------+ +-----------+ +-------------+                   |
                  | (1) SRC   |--+ FMS_Cmd-    |  (4)     | |  (3)      | |   (2)       |                   |
                  | Selector  |--+ Source_     | Mission  | |  Safety   | | Mode        |                   |
                  |           |  | health-     | Sequencer| |  Monitor  | | Manager     |                   |
                  +-----------+  | flags       +----------+ +-----------+ |  (Stateflow)|                   |
                       |         |                  |           |         | (D3 owns)   |                   |
                       |         v                  |           |         +-------------+                   |
                       |   FMS_SrcSel_Bus           |           |                |                          |
                       |   (selected source +       |           |                |                          |
                       |    raw setpoints)          |           |                |                          |
                       +----------+----------+      |           |                |                          |
                                  |          |      |           |                |                          |
                                  v          |      v           |                v                          |
                  +----------------+         |      |           |       FMS_ModeMgr_to_Asm_Bus              |
                  | (3) Safety     |<--------|------|-----------+        (mode/state/ext_state/             |
                  | Monitor        |  failsafe req                       ctrl_mode/error_code/              |
                  +----------------+  -.....->Mode Mgr (dashed)          failsafe_state)                    |
                       |                                                       |                          |
                       | failsafe_req (dashed) → Mode Manager                  |                          |
                       |                                                       |                          |
                       v                                                       |                          |
                  (already merged into Mode Mgr above)                         |                          |
                                                                               |                          |
                                                          FMS_ModeMgr_to_Shape_Bus                        |
                                                          (active mode + ctrl_mode hint +                  |
                                                           setpoint routing instructions)                  |
                                                                |                                          |
                                                                v                                          |
                                                +----------------------+                                   |
                                                |  (5) Command Shaper  |<----- FMS_Mission_to_Shape_Bus    |
                                                |  (D4 owns equations) |       (wp-derived setpoints,      |
                                                |                      |       takeoff/landing trigger)    |
                                                +----------+-----------+                                   |
                                                           |                                               |
                                                           v                                               |
                                                FMS_Shape_to_Asm_Bus                                       |
                                                (shaped setpoints + cmd_mask                               |
                                                 + passthrough-with-shape:                                 |
                                                  actuator_cmd / pilot_thr_pt /                            |
                                                  pilot_yaw_rate_pt)                                       |
                                                           |                                               |
                                                           v                                               |
                                                +----------------------+                                   |
                                                |  (6) Output Assembler|<----- FMS_Passthrough_Direct_Bus  |
                                                |  (FMS_Out_Bus build) |       (wp_*, home_*, ins_*_valid  |
                                                +----------+-----------+        echo) — bypass D4          |
                                                           |                                               |
                                                           v                                               |
                                                       FMS_Out_Bus (out, RB-05 to Controller)              |
                                                                                                           |
                                 +-------------------------------------------------------------------------+
```

**Reset / Init 协调**:per [A6 §4.4.2](../A-architecture/A6-init-reset-contract.md) `FMS_init` → 6 子模块按 §4.8 表回到稳态;运行期 reset 由 Mode Manager 输出的 `reset` 触发(若 B1 锁定该字段存在),全部 6 子模块同步走 local reset(per A6 §4.5.2)。

#### 4.1.2 6 子模块顶层职责一句话

| # | 子模块 | A8 shell(§4.3.2) | 一句话职责 | 内部 Stateflow? | D 区落点 |
|---|---|---|---|---|---|
| (1) | **Source Selector** | `FMS_Source_Selector_Shell` | 把 6 路命令源(Pilot / GCS / Auto / Mission / INS / 可选 Control_Out_Bus)合成单一 `FMS_SrcSel_Bus` + 健康标志,**不**做 mode 决策 | 否(纯组合 + validity gate) | D2 完整;Source 仲裁规则功能层在 D1 §4.2 |
| (2) | **Mode Manager** | `FMS_Mode_Manager_Framework` | 单一 Stateflow,持有 `VehicleStatus` / `VehicleState` / `VehicleExtState` / `FlightMode` / `CtrlMode` 真值;接受 Safety 降级请求与 Mission 进度 | **是**(D3 owns 内部) | D3 详细设计 |
| (3) | **Safety / Failsafe Monitor** | `FMS_Safety_Monitor_Framework` | 评估 [D1 §4.4](D1-fms-functional.md) TR-01..TR-11 的 11 类失效条件;**只**通过 "降级请求" 通知 Mode Manager(per 架构 v1 §12.1.3) | 否(决策树 + 计时器组) | D2 给信号边界;阈值消费 B3 PARAM |
| (4) | **Mission / Auto Manager** | `FMS_Mission_Decoder_Shell` | mission item 推进、wp loiter 计时、pause/continue/abort 状态(`MissionState`);Phase 2 = D1 §4.5 sub-set | 否(顺序状态 + 计时器) | D2 给信号边界;详细 mission 行为分散在 D3 状态 + D4 setpoint + D5 leaf 默认 |
| (5) | **Command Shaper** | `FMS_Command_Shaper_Shell` | 把 Source / Mission 给的原始 setpoint(stick / wp / auto target) 经 mode-条件路由 → 产出 shaped setpoint + cmd_mask 工作位 | 否(mode 索引的查找 + 整形原语级联) | D4 详细设计;D5 leaf 锁多旋翼 profile 常量 |
| (6) | **Output Assembler** | `FMS_Output_Assembler_Shell` | 装配 `FMS_Out_Bus`:shaped setpoints (来自 (5)) + mode/state/ctrl_mode 等(来自 (2)) + passthrough-direct 字段(来自 (1)/(4)) + reset 字段;**不**新生成任何决策 | 否(纯位拷贝 + bus packer) | D2 完整;字段级 schema 引用 B1 §4.7 |

### 4.2 内部信号流(Intra-FMS buses)

per [架构 v1 §10.4 "Internal buses ... module-prefixed"](../../architecture/2026-05-05-fmt-model-architecture-v1.md) — 命名 `FMS_*_Bus`;字段级 schema **不在 D2 范围**(由 D3/D4/D5 各自细化时锁字段)。本节仅枚举**字段类别**与**生产/消费方**。

#### 4.2.1 内部 bus 清单

| Bus 名 | 生产者 | 消费者 | 字段类别(D2 给类别;字段级由下游) | 备注 |
|---|---|---|---|---|
| **`FMS_SrcSel_Bus`** | (1) Source Selector | (2) Mode Manager / (5) Command Shaper / (6) Output Assembler(passthrough-direct 选取) | (a) 选中的 mode 请求源(`pilot_mode_request` / `gcs_mode_request` / `auto_mode_request`); (b) 原始 stick 字段(passthrough);(c) GCS cmd_type + cmd_param;(d) Auto target_pos/vel/acc/yaw(若 mode 为 Offboard);(e) arm_switch / kill_switch latched | 选源结果一次成型;下游不再单独跑仲裁 |
| **`FMS_SrcHealth_Bus`** | (1) Source Selector | (3) Safety Monitor | (a) 每路源的 `valid` flag;(b) 每路源的 staleness(由 `Stale_Detector` A8 §4.2.5 输出);(c) `INS_Status` / `INS_Flag` 直传;(d) `Control_Out_Bus.motor_cmd[]` cross-check 标志(若启用 D1 §4.7 transitional) | Safety 不消费 setpoint 数据,只看健康状态 |
| **`FMS_Safety_to_ModeMgr_Bus`** | (3) Safety Monitor | (2) Mode Manager | (a) `failsafe_request_active`(bool);(b) `failsafe_target_mode`(若 active);(c) `failsafe_state`(B2 §4.4.7 enum);(d) `error_code`(B2 §4.4.10 enum) | 单向降级请求(per 架构 v1 §12.1.3);Mode Manager 仍是唯一 mode 写者 |
| **`FMS_Mission_to_ModeMgr_Bus`** | (4) Mission Manager | (2) Mode Manager | (a) `mission_state`(B2 §4.4.8);(b) `wp_reached`(bool 脉冲);(c) `mission_active`(bool);(d) `mission_done`(bool) | Mode Manager 在 M-09 / M-10 中据此推 mode |
| **`FMS_Mission_to_Shape_Bus`** | (4) Mission Manager | (5) Command Shaper | (a) `wp_target_pos_ned_m[3]`(从当前 wp 派生);(b) `wp_target_yaw_rad`(若 wp 含);(c) takeoff/landing/RTL phase 状态(枚举 `MISSION_PHASE_*`,Mission internal,**非** B2 enum);(d) mission default speed(echo from PARAM .27) | wp 由 Mission 负责"选取并派生坐标",Shaper 不做 wp 数据库管理 |
| **`FMS_ModeMgr_to_Shape_Bus`** | (2) Mode Manager | (5) Command Shaper | (a) `active_mode`(B2 `FlightMode` 数值);(b) `ctrl_mode_hint`(B2 `CtrlMode`);(c) setpoint 路由指令(`route_pos_from`(enum:HOLD/MISSION/PILOT_OFFSET/AUTO_TARGET)/ `route_vel_from` / `route_att_from` / `route_rate_from` / `route_throttle_from`);(d) shaping enable 位(每整形原语一位:`enable_jerk_lim` / `enable_tilt_lim` / `enable_takeoff_profile` / 等) | Mode Manager 决定"在该 mode 下 shaper 走哪条路";Shaper 不再判 mode |
| **`FMS_ModeMgr_to_Asm_Bus`** | (2) Mode Manager | (6) Output Assembler | (a) `mode`(B2 `FlightMode`);(b) `status` / `state` / `ext_state`(B2 §4.4.1/2/3);(c) `ctrl_mode`(B2 §4.4.6);(d) `failsafe_state`(B2 §4.4.7);(e) `error_code`(B2 §4.4.10);(f) `reset` bool(若 B1 锁定该字段;global/local 由 A6 §4.5.2)| 这些字段 Output Assembler 直接写入 `FMS_Out_Bus`,不做加工 |
| **`FMS_Shape_to_Asm_Bus`** | (5) Command Shaper | (6) Output Assembler | (a) shaped setpoints 全集(`pos_cmd_ned_m` / `vel_cmd_ned_mps` / `acc_cmd_ned_mps2` / `att_cmd_quat` / `att_cmd_euler_rad` / `ang_rate_cmd_b_radps` / `yaw_cmd_rad` / `yaw_rate_cmd_radps` / `throttle_cmd`);(b) `cmd_mask`(uint32 bitfield;**位语义** D6 锁);(c) passthrough-with-shape 字段(per A3 §4.6 双类:`actuator_cmd[]` / `pilot_throttle_passthrough` / `pilot_yaw_rate_passthrough_radps`)| 字段名集与 A3 §4.4.6.1 / B1 §4.7 ledger 对齐;具体字段顺序由 B1 锁 |
| **`FMS_Passthrough_Direct_Bus`** | (1) Source Selector + (4) Mission Manager | (6) Output Assembler | passthrough-direct(per A3 §4.6 表 — 不经 D4 Shaper):(a) `wp_lat_deg` / `wp_lon_deg` / `wp_alt_m`;(b) `home_lat_deg` / `home_lon_deg` / `home_alt_m`;(c) `wp_index`;(d) `motor_cmd_passthrough[]`(若 variant 启用);(e) `ins_ready` / `ins_position_valid` / `ins_attitude_valid`(若 B1 锁定 echo 字段) | A3 §4.6 表里类别 = passthrough-direct 的字段全部走此路径;不**经 D4 Shaper 任何整形** |

#### 4.2.2 信号流的"职责正交性"约束

每条内部 bus 仅承载**一种类别**的信息,以避免子模块把跨域职责挤进同一管道:

1. **不**允许 Source Selector 直接把 setpoint 喂给 Output Assembler — 必须经 Command Shaper(原因:即便 Manual mode 的 stick 也要经死区 / 单位换算,per [A3 §4.6 passthrough-with-shape](../A-architecture/A3-module-boundaries.md)/[D1 §4.3.1](D1-fms-functional.md))。
2. **不**允许 Safety Monitor 直接写 `FMS_Out_Bus.mode` — 必须经 Mode Manager(per 架构 v1 §12.1.3 "can request mode degradation through the Mode Manager only")。
3. **不**允许 Mission Manager 跨过 Mode Manager 切 mode — Mission 只发"wp_reached / mission_done",Mode Manager 据此切。
4. **不**允许 Command Shaper 决定 mode — Shaper 只接收 `active_mode` 与 routing 指令,自身无 mode 状态。
5. **passthrough-direct vs passthrough-with-shape 严格分流**(per A3 §4.6 双类):前者走 `FMS_Passthrough_Direct_Bus` 直入 Output Assembler;后者走 `FMS_Shape_to_Asm_Bus` 经 D4 整形。

### 4.3 子模块边界

#### 4.3.1 (1) Source Selector

| 项 | 内容 |
|---|---|
| **A8 shell** | [`FMS_Source_Selector_Shell`](../A-architecture/A8-shared-library-roster.md) (Layer B FMS shared) |
| **Owner** | `model/fms/shared/`(per A5 多机型共享;D5 leaf 不重写) |
| **输入** | `Pilot_Cmd_Bus` / `GCS_Cmd_Bus` / `Auto_Cmd_Bus` / `Mission_Data_Bus` / `INS_Out_Bus` / 可选 `Control_Out_Bus`(D1 §4.7 transitional) — 全部为 [A4 RB-06](../A-architecture/A4-rate-boundaries.md) latch / [RB-04](../A-architecture/A4-rate-boundaries.md) ZOH 后副本 |
| **输出** | `FMS_SrcSel_Bus`(到 Mode Manager / Command Shaper / Output Assembler 的 passthrough-direct 选取) + `FMS_SrcHealth_Bus`(到 Safety Monitor) |
| **核心职责** | (a) 每路源 validity + staleness 检查([A8 §4.2.5 `Stale_Detector`](../A-architecture/A8-shared-library-roster.md) + [`Validity_AND`](../A-architecture/A8-shared-library-roster.md) 复用);(b) 按 [D1 §4.2.2 仲裁优先级](D1-fms-functional.md)选源(SAFETY 优先级在 Mode Manager 层落地,本子模块只产生"源选择候选" + 健康标志);(c) `INS_Status` / `INS_Flag` 直传(为 health bus 与 passthrough-direct echo 字段服务);(d) 为 Output Assembler 的 passthrough-direct 字段(`home_*` from GCS set_home / `wp_*` index hint) 提供 selection |
| **Reset** | per [A6 §4.4.2](../A-architecture/A6-init-reset-contract.md):`Stale_Detector` 计时器复位;每路 `valid` reset 后默认 false 直至首次有效注入;`FMS_SrcSel_Bus` 字段 reset 后类别 = ZERO / HARDCODED(per A6 表);**不持有** mode 状态,无需 mode-reset |
| **速率** | 20 ms(FMS 内部统一,per [架构 v1 §13](../../architecture/2026-05-05-fmt-model-architecture-v1.md)) |
| **Stateflow?** | 否(组合逻辑 + 计时器) |
| **D 区下游 owner** | D2 完整规定边界;源仲裁功能层 D1 §4.2;**不**进 D3/D4/D5 |

#### 4.3.2 (2) Mode Manager

| 项 | 内容 |
|---|---|
| **A8 shell** | [`FMS_Mode_Manager_Framework`](../A-architecture/A8-shared-library-roster.md) (Layer B FMS shared,Stateflow chart 骨架) |
| **Owner** | `model/fms/shared/`(per A5;D5 leaf 仅注入"多旋翼专属转换"通过 PARAM 守卫,不重写 chart 拓扑) |
| **输入** | `FMS_SrcSel_Bus`(选中源 + mode 请求) + `INS_Out_Bus`(直消费 INS validity 位作为转换守卫,per A3 §4.4.4) + `FMS_Safety_to_ModeMgr_Bus`(failsafe 降级请求) + `FMS_Mission_to_ModeMgr_Bus`(mission 进度) |
| **输出** | `FMS_ModeMgr_to_Shape_Bus`(到 Command Shaper) + `FMS_ModeMgr_to_Asm_Bus`(到 Output Assembler) |
| **核心职责** | (a) 持有唯一的 `VehicleStatus` / `VehicleState` / `VehicleExtState` / `FlightMode` / `CtrlMode` 真值(per 架构 v1 §12.1.2);(b) 处理 D1 §4.1 12 主 mode + 4 failsafe sub-mode;(c) 仲裁 SAFETY override → PILOT mode_switch → GCS → AUTO → MISSION 优先级(D1 §4.2.2);(d) 转换守卫消费 INS gate 位 + arm 系列 PARAM(B3 .02..05);(e) 给 Command Shaper 发"setpoint 路由指令" + shaping enable 位(per §4.2.1 `FMS_ModeMgr_to_Shape_Bus`);(f) 派生 `error_code`(B2 §4.4.10);(g) 生成 `reset` 字段(若 B1 锁定其存在,per A6 §4.5.2 触发 local reset)|
| **Reset** | per [A6 §4.4.2](../A-architecture/A6-init-reset-contract.md):mode reset 后 = `MODE_NONE`;status = `STATUS_DISARM`;state = `STATE_DISARM`;ext_state = `EXT_STATE_LANDED`;ctrl_mode = `CMODE_DISARMED`;**Stateflow chart entry state** = D3 owns(per D1 §4.1.3 已选 M-01 IDLE);chart 内部计时器全清 |
| **速率** | 20 ms |
| **Stateflow?** | **是** — 单一 chart;状态/转换/事件/守卫由 [D3 详细设计](D3-mode-manager.md) 锁定(Wave 8 co-seal) |
| **D 区下游 owner** | **D3** owns 内部状态机;D2 仅给边界 + 期望输入/输出 |

#### 4.3.3 (3) Safety / Failsafe Monitor

| 项 | 内容 |
|---|---|
| **A8 shell** | [`FMS_Safety_Monitor_Framework`](../A-architecture/A8-shared-library-roster.md) (Layer B FMS shared) |
| **Owner** | `model/fms/shared/`;D5 leaf 仅传入 PARAM 阈值(per [B3 §4.4 FMS_PARAM .11..29](../B-contracts/B3-parameter-schema.md) `geofence_*` / `failsafe_*` / `low_battery_*`) |
| **输入** | `FMS_SrcHealth_Bus`(健康标志 + INS status / flag) + `INS_Out_Bus`(直消费 `position_ned_m` / `position_lla.alt_m` for geofence 检测,per A3 §4.4.4) + `Pilot_Cmd_Bus.kill_switch`(直读,per D1 TR-10 immediate) + 可选 `Control_Out_Bus`(D1 §4.7 transitional cross-check;TBD H4 启用) + 电池电压字段 *(D1 §5 open;源未定;Phase 2 placeholder)* |
| **输出** | `FMS_Safety_to_ModeMgr_Bus`(降级请求 + failsafe_state + error_code) |
| **核心职责** | 评估 [D1 §4.4 TR-01..TR-11](D1-fms-functional.md) 11 类触发,按 D1 §4.4.2 优先级裁出最高一条;消费 [`Stale_Detector`](../A-architecture/A8-shared-library-roster.md) / [`Threshold_Latch`](../A-architecture/A8-shared-library-roster.md) / `Validity_AND` 等 Layer A 原语组合;debounce / hysteresis(per D1 §4.4.3)由本子模块计时器组承担 |
| **Reset** | per A6 §4.4.2:`failsafe_state` reset 后 = `FAIL_NONE`;`error_code` = `ERR_NONE`;所有 debounce 计时器清零;**不**持有 mode 状态 |
| **速率** | 20 ms |
| **Stateflow?** | 否(决策树 + 计时器组,Simulink 块 + Layer A 原语) |
| **D 区下游 owner** | D2 完整规定结构边界;阈值消费 B3 PARAM;TR-05/06 电池来源 open(D1 §5)由 H4 协同决定 |

#### 4.3.4 (4) Mission / Auto Manager

| 项 | 内容 |
|---|---|
| **A8 shell** | [`FMS_Mission_Decoder_Shell`](../A-architecture/A8-shared-library-roster.md) (Layer B FMS shared);消费 Layer A [`Mission_Item_Decoder`](../A-architecture/A8-shared-library-roster.md) |
| **Owner** | `model/fms/shared/`;D5 leaf 锁多旋翼默认任务速度 / wp 接受半径 / wp loiter 默认时长(per B3 PARAM .25..27) |
| **输入** | `FMS_SrcSel_Bus.mission_data`(选中的 `Mission_Data_Bus`) + GCS pause/continue/abort 命令(via `FMS_SrcSel_Bus.gcs_cmd_type` ∈ `CMDTYPE_PAUSE_MISSION` / `_RESUME_MISSION`)+ `FMS_ModeMgr_to_Shape_Bus.active_mode`(知道何时 active) + `INS_Out_Bus.position_ned_m`(用于 wp 到达判定) |
| **输出** | `FMS_Mission_to_ModeMgr_Bus`(mission_state / wp_reached / mission_active / mission_done) + `FMS_Mission_to_Shape_Bus`(wp 派生 setpoint 候选 + takeoff/landing/RTL phase 内部状态) |
| **核心职责** | (a) 保存当前 wp 索引,推进顺序(D1 §4.5.1);(b) 计算 wp 到达(距离 ≤ `wp_acceptance_radius_m` PARAM .26);(c) 维护 `MissionState`(B2 §4.4.8);(d) takeoff / landing / RTL 三个 internal sequence(per D1 §4.3.4),把"现在该飞到哪"传给 Shaper;(e) Phase 2 cut(D1 §4.5.2 排除项)对应字段 disable;(f) Mission abort 由 Safety 触发(经 Mode Manager 切 RTL/LAND,Mission 收到 active_mode 变化后转 ABORTED) |
| **Reset** | per A6 §4.4.2:`mission_state` reset 后 = `MISSION_IDLE`;`wp_index` = HARDCODED 0(或 PRESERVED — 由 B1 锁定 §4.4.6.3);takeoff/landing/RTL phase 内部状态 = phase_0;若 A6 §5 标 PRESERVED 类(boot 之间),则 init 后保留上次值,reset 时仍清零 |
| **速率** | 20 ms |
| **Stateflow?** | 否(顺序状态枚举 + 计时器);Mission Sequencer 用 truth-table / chart-style block,但**不**与 Mode Manager 同 chart |
| **D 区下游 owner** | D2 完整规定边界;mission 范围 D1 §4.5;wp 内 setpoint 派生公式分散到 D4(takeoff/landing 经 Shaper)与 D5(默认时长 / 速度) |

#### 4.3.5 (5) Command Shaper

| 项 | 内容 |
|---|---|
| **A8 shell** | [`FMS_Command_Shaper_Shell`](../A-architecture/A8-shared-library-roster.md) (Layer B FMS shared);消费 Layer A [`Rate_Limiter_3d`](../A-architecture/A8-shared-library-roster.md) / [`Jerk_Limiter_3d`](../A-architecture/A8-shared-library-roster.md) / [`Deadzone_With_Linear_Bridge`](../A-architecture/A8-shared-library-roster.md) |
| **Owner** | `model/fms/shared/`(macro 共享);D5 leaf 锁多旋翼专属 profile 数值 / 限值 |
| **输入** | `FMS_SrcSel_Bus`(stick / Auto target / GCS setpoint update) + `FMS_ModeMgr_to_Shape_Bus`(active_mode + ctrl_mode_hint + 路由指令 + shaping enable 位) + `FMS_Mission_to_Shape_Bus`(wp-derived setpoint 候选 + takeoff/landing phase) + 可选 PARAM 引用(B3 .33..41 limits)|
| **输出** | `FMS_Shape_to_Asm_Bus`(shaped setpoints + cmd_mask 工作位 + passthrough-with-shape 字段) |
| **核心职责** | (a) 按 routing 指令把"原始 setpoint 候选"路由到对应字段;(b) D1 §4.3 整形(死区 / expo / rate / jerk / tilt / yaw_rate 限);(c) 起飞 / 降落 / RTL profile 数值生成(D1 §4.3.4 类别 → D4 公式 → D5 leaf 数值);(d) 设置 `cmd_mask` 工作位(per D1 §4.6.2;**位语义** D6/E1 Wave 8);(e) passthrough-with-shape 字段(`actuator_cmd[]` / `pilot_throttle_passthrough` / `pilot_yaw_rate_passthrough_radps`,per A3 §4.6 表)经本子模块整形(死区 + 单位换算 + 限幅) |
| **Reset** | per A6 §4.4.2:全部 setpoint 字段 reset 后 = ZERO;`cmd_mask` reset 后 = 全 0(safe state);Rate/Jerk Limiter 内部状态(上一帧值)清零;Takeoff/Landing 内部 profile 计时器清零;passthrough-with-shape 字段 reset 后 = ZERO |
| **速率** | 20 ms |
| **Stateflow?** | 否(mode-索引的查找 + 整形原语级联;D5 中可有 mode-conditional 子图但非 chart) |
| **D 区下游 owner** | **D4** owns 整形公式;**D5** owns 多旋翼数值参数;D2 仅给边界 |

#### 4.3.6 (6) Output Assembler

| 项 | 内容 |
|---|---|
| **A8 shell** | [`FMS_Output_Assembler_Shell`](../A-architecture/A8-shared-library-roster.md) (Layer B FMS shared) |
| **Owner** | `model/fms/shared/`(macro 共享);D5 leaf 不参与(字段集由 B1 锁) |
| **输入** | `FMS_Shape_to_Asm_Bus`(shaped setpoints + cmd_mask + passthrough-with-shape) + `FMS_ModeMgr_to_Asm_Bus`(mode/state/ext_state/ctrl_mode/failsafe_state/error_code/reset) + `FMS_Passthrough_Direct_Bus`(wp_*/home_*/ins_*_valid echo + motor_cmd_passthrough variant) |
| **输出** | `FMS_Out_Bus`(单一边界输出,per [架构 v1 §8.3](../../architecture/2026-05-05-fmt-model-architecture-v1.md);字段级 schema [B1 §4.7](../B-contracts/B1-bus-inventory.md)) |
| **核心职责** | (a) 字段级位拷贝 + bus packing;(b) 不引入新逻辑;(c) `timestamp` 字段直接来自 step interface(per [A7 §4.6](../A-architecture/A7-time-conventions.md));(d) 处理 [A3 §4.6 passthrough 双类](../A-architecture/A3-module-boundaries.md) — passthrough-with-shape 已在 D4 处理后入此装配,passthrough-direct 直入装配;(e) reset 字段 echo from Mode Manager;(f) 验证 enum-typed 字段值在 B2 锁定的合法集内(可选 assertion,debug-only) |
| **Reset** | per [A6 §4.4.2](../A-architecture/A6-init-reset-contract.md) 完整表 — Output Assembler 在 reset 时把 `FMS_Out_Bus` 全部字段写为表中规定的稳态值类别(PARAM / HARDCODED / ZERO / PRESERVED);**这是 D2 中唯一**写 `FMS_Out_Bus` 的子模块,因此 reset 一致性**集中在此**保证 |
| **速率** | 20 ms;输出经 [A4 §4.4 RB-05](../A-architecture/A4-rate-boundaries.md) latch 到 Controller 5 ms |
| **Stateflow?** | 否(纯 bus packer + multiplexer) |
| **D 区下游 owner** | D2 完整规定边界;字段级 schema B1 §4.7 |

### 4.4 Mode Manager 到 D3 的 hand-off

D2 把以下事项**冻结**作为 D3 启动输入(D3 详细设计 Wave 8 co-seal):

| Hand-off 项 | D2 已锁(本文件 §) | D3 owns |
|---|---|---|
| Mode Manager 子模块**单一 Stateflow chart** 实现(per 架构 v1 §12.1.2) | §4.1.2 (2) + §4.3.2 | chart 内部状态层级 / 子状态(若有) |
| 状态 = D1 §4.1 mode 集 — 12 主 mode + 4 failsafe sub-mode 全集 | §4.3.2 输入/输出 | 每个状态的 `entry` / `during` / `exit` 动作 |
| Stateflow 输入接口 | §4.2.1 `FMS_SrcSel_Bus` + `INS_Out_Bus` validity 位 + `FMS_Safety_to_ModeMgr_Bus` + `FMS_Mission_to_ModeMgr_Bus` | 每个事件 / 守卫的具体表达式(消费 D1 §4.1.4 转换矩阵 + B3 PARAM 守卫数值) |
| Stateflow 输出接口 | §4.2.1 `FMS_ModeMgr_to_Shape_Bus`(active_mode + ctrl_mode_hint + 路由指令 + shaping enable 位) + `FMS_ModeMgr_to_Asm_Bus`(mode/state/ext_state/ctrl_mode/failsafe_state/error_code/reset)| 每个状态向 routing 指令的具体取值(把 D1 §4.2.4 mode-source 喂养表落到 routing 字段);chart 内的 ctrl_mode 取值落实(D1 §4.1.1 echo) |
| Reset 行为 | §4.3.2 Reset 行 — 走 A6 §4.4.2 类别;**进入 init 状态** = M-01 IDLE | chart 的 initial transition 与 reset action |
| 与 Mission Sequencer 的解耦 | §4.2.1 `FMS_Mission_to_ModeMgr_Bus` 字段类别 — Mission 仅发"wp_reached / mission_done";Mode Manager 据此推 mode | Mode Manager 内部如何消费 wp_reached(进入 LOITER vs 推下个 wp 的 chart 转换) |
| 与 Safety 的解耦 | §4.2.1 `FMS_Safety_to_ModeMgr_Bus` 字段类别 — Safety 仅发 `failsafe_request_active` + `failsafe_target_mode` + `failsafe_state` + `error_code` | Mode Manager 内部 SAFETY 优先级生效落地(架构 v1 §12.1.3 "通过 Mode Manager only") |

**D3 不需要从 D2 重新论证**:6 子模块拓扑(D1 §4.8 已锁)、Mode Manager 是单一 chart 的事实(架构 v1 §12.1.2 + 本文件)、Stateflow 输入/输出 bus 字段类别(本文件 §4.2)、reset entry state(D1 §4.1.3 已锁)。

### 4.5 Command Shaper 到 D4 的 hand-off

D2 把以下事项**冻结**作为 D4 启动输入(D4 详细设计 Wave 8 co-seal):

| Hand-off 项 | D2 已锁(本文件 §) | D4 owns |
|---|---|---|
| Command Shaper 子模块**单一 subsystem** 实现 | §4.1.2 (5) + §4.3.5 | subsystem 内部 mode-conditional 路由的 Simulink 实现(switch / multiport switch / variant subsystem 均可) |
| Shaper 输入 | §4.2.1 `FMS_SrcSel_Bus` + `FMS_ModeMgr_to_Shape_Bus` + `FMS_Mission_to_Shape_Bus` + B3 PARAM 引用 | shaper 内部 routing 树(per active_mode + 路由指令)+ 整形原语级联(每条 mode-source 路径) |
| Shaper 输出 | §4.2.1 `FMS_Shape_to_Asm_Bus` 字段类别(shaped setpoints + cmd_mask 工作位 + passthrough-with-shape) | 每个字段在每个 mode 的具体来源链路(D1 §4.2.4 表 → 整形原语 → 输出);cmd_mask 工作位的"在哪个 mode 置位"(语义留 D6) |
| 整形原语清单 | §4.3.5 引用 A8 §4.2 / §4.3 — `Rate_Limiter_3d` / `Jerk_Limiter_3d` / `Deadzone_With_Linear_Bridge` / `LowPass_Filter_1st`(可选 stick smoothing) | 每条 mode-source 链路上**用哪些原语 + 串接顺序**;具体 PARAM 接线(B3 .33..41) |
| 起飞 / 降落 / RTL profile | §4.3.5 + D1 §4.3.4 类别 + Mission Manager 在 §4.2.1 给的 phase 状态 | 三个 profile 的数学公式(类别)+ 与 Mission phase 状态联动;具体数值(初值、爬升速度、touch-down 速度、disarm dwell)由 D5 leaf |
| Reset | §4.3.5 Reset 行 | Rate/Jerk Limiter 内部状态 reset 数学(per E3 风格);profile 计时器初始化策略 |
| passthrough-with-shape 三字段 | §4.2.1 `FMS_Shape_to_Asm_Bus` 包含 `actuator_cmd[]` / `pilot_throttle_passthrough` / `pilot_yaw_rate_passthrough_radps`(per A3 §4.6 表);D2 锁"经 Shaper" | shaper 对这三字段的整形细节(死区 / 单位换算 / 限幅 / 仅 manual mode 启用 etc.) |

**D4 不需要从 D2 重新论证**:Shaper 是单一 subsystem(本文件)、消费什么 / 产出什么(本文件 §4.2)、6 子模块拓扑(D1 §4.8)。

### 4.6 Output Assembler 结构

per [架构 v1 §12.1.6](../../architecture/2026-05-05-fmt-model-architecture-v1.md) "populate `FMS_Out_Bus`" + [D1 §4.6](D1-fms-functional.md) FMS 产出字段类别 + [A3 §4.6 passthrough 双类](../A-architecture/A3-module-boundaries.md):

#### 4.6.1 字段路由分类

Output Assembler 把 `FMS_Out_Bus`(B1 §4.7 ledger)字段集按"来源路径"分为四类:

| 字段路径类别 | 来源 bus | 在 D2 拓扑中的路径 | 经 D4 Shaper? |
|---|---|---|---|
| **Setpoint(D1 §4.6.1)** | `FMS_Shape_to_Asm_Bus` | (5) Command Shaper → (6) Output Assembler | 是(setpoint 字段全部由 D4 整形产出) |
| **mode/state/ctrl_mode/error 组(D1 §4.6.2)** | `FMS_ModeMgr_to_Asm_Bus` | (2) Mode Manager → (6) Output Assembler | 否(状态枚举不需要整形) |
| **passthrough-with-shape(A3 §4.6 第 1 类)** | 来自 `FMS_SrcSel_Bus` 中 stick/cmd_param,经 (5) Shaper 整形 | (1) Source Selector → (5) Shaper → (6) Output Assembler | **是**(per A3 §4.6 = `actuator_cmd[]` / `pilot_throttle_passthrough` / `pilot_yaw_rate_passthrough_radps`) |
| **passthrough-direct(A3 §4.6 第 2 类)** | `FMS_Passthrough_Direct_Bus`(由 (1) Source Selector + (4) Mission 联合产生) | (1)/(4) → (6) Output Assembler **直接** | **否**(per A3 §4.6 = `wp_*` / `home_*` / `motor_cmd_passthrough[]` variant / `ins_*_valid` echo) |

**关键不变量**:每个 `FMS_Out_Bus` 字段属且仅属一个上述类别;[A3 §4.6 表](../A-architecture/A3-module-boundaries.md)是单一来源,B1 §4.7 字段顺序由 byte-level 锁定。

#### 4.6.2 cmd_mask 编排

`cmd_mask` 由 (5) Command Shaper 设置 → 经 `FMS_Shape_to_Asm_Bus` 进入 (6) Output Assembler → 写入 `FMS_Out_Bus.cmd_mask`。

**D2 不锁**位语义(D6/E1 Wave 8 co-seal),但**锁**:
- cmd_mask 的**生产者**是 Command Shaper(单一写者)
- Output Assembler 不修改 cmd_mask 位
- Reset 时 cmd_mask = 全 0(per A6 §4.4.2 + D1 §4.1.3)
- cmd_mask 的工作位号(B2 §4.4.14)由 [D6](D6-fms-controller-interface.md) 锁定

#### 4.6.3 timestamp 字段

per [A7 §4.6](../A-architecture/A7-time-conventions.md):`FMS_Out_Bus.timestamp` 直接 echo `FMS_step` 入参 timestamp;Output Assembler **不**重新打戳。

#### 4.6.4 reset 字段(若 B1 锁定其存在)

per [A6 §4.5.2 global vs local reset](../A-architecture/A6-init-reset-contract.md):若 B1 §4.7 锁 `FMS_Out_Bus.reset` 字段存在 → Output Assembler 直传 Mode Manager 给的 `reset` 信号;global reset 由 Harness 触发(per A6 §4.5.2 关键约束 1,Plant 不联动)。

### 4.7 变体策略(per A5)

per [A5 Phase 2 = 多旋翼唯一 leaf](../A-architecture/A5-variant-strategy.md):

| 项 | D2 决议 |
|---|---|
| **6 子模块的归属** | 6 个**全部 shared**(`model/fms/shared/`);per [A8 §4.3.2](../A-architecture/A8-shared-library-roster.md) 7 个 Layer B FMS shell 与之对应(注:`FMS_Reset_Hooks_Shell` 是横切 hook,不单独成子模块,作为 reset 流水线注入子模块) |
| **D5 多旋翼 leaf 注入点** | (a) Command Shaper 内部:多旋翼专属 takeoff/landing/RTL profile 数值常量(per D1 §4.3.4);(b) Source Selector 内部:多旋翼专属 stick mapping(roll/pitch → tilt 角的最大值,per B3 PARAM .38 `tilt_lim_rad`);(c) Mission Manager 内部:多旋翼专属默认任务速度 / wp loiter 默认时长(B3 PARAM .27 / .26);(d) Mode Manager Stateflow 内部:多旋翼专属 entry guard(例:`PMODE_OFFBOARD` 进入需 INS pos+att valid;ACRO sub-branch per D5 决定是否启用 M-12) |
| **leaf 注入机制** | 通过 [B3 `FMS_PARAM`](../B-contracts/B3-parameter-schema.md) 字段(per A5 + D1 §4.8.3 leaf 重写授权);**不**通过 Variant Subsystem 切换;**不**重写 chart 拓扑 |
| **Phase 5+ 其他机型(fixed-wing / VTOL / 等)** | 每个机型 = 独立 leaf 文件(per A5);D2 macro 拓扑(6 子模块 + 内部 bus)**复用,不变**;新 leaf 仅注入 PARAM 与 mode-source 喂养表的机型行(D1 §4.2.4 类别可扩) |
| **过度抽象预算上限**(per A5) | D2 6 子模块全部归 shared;leaf 文件数 = 1 per vehicle(只有 D5 形式的 PARAM + profile 常量集);若未来 leaf 文件数超过 A5 锁定的预算上限,需重新审议 D2 切分 |

### 4.8 Init / Reset(per A6)

#### 4.8.1 `FMS_init` 子模块协议

per [A6 §4.2 / §4.3](../A-architecture/A6-init-reset-contract.md):

| 子模块 | `FMS_init` 时动作 |
|---|---|
| (1) Source Selector | `Stale_Detector` 计时器清零;每路 `valid` HARDCODED false 直至首次注入;`FMS_SrcSel_Bus` 字段稳态值类别 = ZERO / HARDCODED |
| (2) Mode Manager | Stateflow chart 进入 initial state(D3 锁)= M-01 IDLE;`mode` = `MODE_NONE` / `status` = `STATUS_DISARM` / `state` = `STATE_DISARM` / `ext_state` = `EXT_STATE_LANDED` / `ctrl_mode` = `CMODE_DISARMED`(per [A6 §4.4.2](../A-architecture/A6-init-reset-contract.md) + [D1 §4.1.3](D1-fms-functional.md));chart 内部计时器清零 |
| (3) Safety Monitor | `failsafe_state` = `FAIL_NONE`;`error_code` = `ERR_NONE`;debounce 计时器全清;PARAM .11..29 由 B3 init payload 加载 |
| (4) Mission Manager | `mission_state` = `MISSION_IDLE`;`wp_index` = HARDCODED 0(或 PRESERVED 由 B1 锁);takeoff/landing/RTL phase 状态 = phase_0;PARAM .25..27 加载 |
| (5) Command Shaper | 全部 setpoint 字段稳态 = ZERO;`cmd_mask` = 全 0;Rate/Jerk Limiter 内部状态(上一帧值)清零;Takeoff/Landing 内部计时器清零;PARAM .33..41 加载 |
| (6) Output Assembler | `FMS_Out_Bus` 全部字段写为 [A6 §4.4.2](../A-architecture/A6-init-reset-contract.md) 表中规定的稳态值类别(PARAM / HARDCODED / ZERO / PRESERVED);timestamp = 0 |

#### 4.8.2 运行期 reset

per [A6 §4.5.2 global vs local reset](../A-architecture/A6-init-reset-contract.md):

| 触发 | 范围 | 子模块响应 |
|---|---|---|
| Harness global reset | 全模块 | 等同 `FMS_init`;6 子模块全部回 init 后稳态(§4.8.1);Output Assembler 写表中规定值 |
| Mode Manager local reset(由 D3 chart action 触发,例:disarm / lockdown 重新 arm) | 部分 | (1)/(3)/(4)/(5) 视具体 chart action 决定;一致约束:[A6 §4.4.2](../A-architecture/A6-init-reset-contract.md) reset 类别表的字段必须回到表中规定值 — 此一致性由 (6) Output Assembler 集中保证(§4.6.4) |

#### 4.8.3 INS readiness gate

per [A6 §4.5.3](../A-architecture/A6-init-reset-contract.md) + [D1 §4.1.3](D1-fms-functional.md):

- `FMS_init` 完成后,Mode Manager **不**自动离开 M-01 IDLE,必须等 `INS_Out_Bus.INS_Status.bit.ready==1` 且 pilot `arm_switch==1` 且 `arm_throttle_thr_n01`(B3 PARAM .02)守卫满足
- 在 INS not ready 期间,Source Selector 仍把 `INS_Out_Bus` 直传给 Safety Monitor 与 Mode Manager(让它们看到 `bit.ready==0`),但 Command Shaper 在 INS-依赖 mode(M-04..M-11)的路由指令为 invalid → shaper 不会输出 INS-依赖 setpoint(失败安全)
- 此 INS gate 守卫由 D3 chart 的 transition guard 落地;D2 仅保证信号到达 Mode Manager

### 4.9 跨引用与下游

D2 是 D 区结构起点(D1 是功能起点);本工作项之后 D 区其余文件的结构输入均来自本文件:

| 下游 | 关系类型 | D2 提供 |
|---|---|---|
| [D3 Mode Manager](D3-mode-manager.md)(Wave 8 co-seal) | → | §4.3.2 + §4.4 — Mode Manager 子模块边界 + Stateflow 输入/输出 bus 字段类别 + reset entry state + 单一 chart 约束 |
| [D4 Command Shaper](D4-command-shaper.md)(Wave 8 co-seal) | → | §4.3.5 + §4.5 — Shaper 子模块边界 + `FMS_Shape_to_Asm_Bus` 字段类别 + routing 指令接口 + 整形原语清单引用 |
| [D5 多旋翼 leaf](D5-multicopter-leaf.md)(Wave 9) | → | §4.7 — 6 子模块全 shared;leaf 注入点 = PARAM-driven 4 处(Shaper profile / Source Selector stick / Mission default / Mode Manager guard) |
| [D6 FMS↔Controller 接口](D6-fms-controller-interface.md)(Wave 8 co-seal) | ↔ | §4.6.2 — cmd_mask 由 Command Shaper 单一写,Output Assembler 不修改;位语义留 D6;cmd_mask reset = 全 0 |
| [E1 Controller 功能](../E-controller/E1-controller-functional.md)(Wave 8 co-seal) | ⇢ | §4.2.1 `FMS_Shape_to_Asm_Bus` 字段类别(经 Output Assembler → `FMS_Out_Bus`)消费侧 |
| [F3 INS 消费规则](../F-ins-contract/F3-consumption-rules.md)(Wave 9) | ⇢ | §4.3.2 / §4.3.3 / §4.3.5 — INS_Out_Bus 在 Mode Manager / Safety / Shaper 三处消费的字段类别(per A3 §4.4.4 矩阵) |
| [G1 MIL 顶层](../G-harness/G1-mil-toplevel.md)(Wave 10) | ⇢ | §4.1.1 顶层框图 + §4.7 6 子模块全 shared = G1 把 FMS 作为单一 reference model 接入(macro 不切分到顶层) |
| [G2 多速率调度](../G-harness/G2-rate-scheduling.md)(Wave 10) | ⇢ | §4.3 各子模块 "速率 = 20 ms" + 输出经 RB-05 latch — G2 在 FMS 边界配置 RB-05 |

## 5. 已知风险与悬而未决问题

- **`reset` 字段是否在 `FMS_Out_Bus` 中存在(B1 锁)**
  - 影响:若 B1 锁定该字段不在 bus 中,则 §4.6.4 与 §4.3.2 Mode Manager 输出中的 `reset` 信号变成 FMS-internal 状态(只在 6 子模块之间传递,不出 bus 边界)
  - 处置:open;由 [B1 §4.7](../B-contracts/B1-bus-inventory.md) byte-level 表锁定;若不在 bus 中,本文件 §4.6.4 + §4.2.1 `FMS_ModeMgr_to_Asm_Bus` 字段类别 (f) 行随之标 "internal-only"
- **`Control_Out_Bus` 在 D2 中的物理路径未细化(D1 §4.7 transitional)**
  - 影响:D1 §4.7 已锁 transitional + Safety cross-check 用途;D2 §4.3.3 Safety Monitor 输入项标 "可选 `Control_Out_Bus`(TBD H4 启用)" 但具体 cross-check 算法 / 阈值未定
  - 处置:open;[H4 故障注入目录](../H-verification/H4-fault-catalog.md) 协同决定 "controller silent fault" 检测器是否启用 + 阈值;若启用,Safety Monitor 内部增一条 TR-12 触发(D1 §4.4.1 表外)
- **passthrough-direct vs passthrough-with-shape 字段集需 B1 锁**
  - 影响:§4.6.1 双类划分依赖 [A3 §4.6 表](../A-architecture/A3-module-boundaries.md);若 B1 锁定后 `motor_cmd_passthrough[]` 不在 `FMS_Out_Bus` 中(per A3 §4.4.6.4 备注 "多旋翼 Phase 2 默认禁用"),`FMS_Passthrough_Direct_Bus` 字段类别 (d) 行随之 disable
  - 处置:open;由 B1 锁;D2 中字段名只用类别名,不字段级,因此 B1 变更不破坏本文件结构
- **Mission Sequencer 与 Mode Manager 的"谁负责 takeoff/landing/RTL phase 推进"边界**
  - 影响:§4.2.1 `FMS_Mission_to_Shape_Bus` 字段类别 (c) "takeoff/landing/RTL phase" 由 Mission Manager 持有;但 D1 §4.1 中 M-06/M-07/M-08 是 Mode Manager 的状态(不是 Mission 的)— 物理推进谁拥有?D2 折衷为 "Mode Manager 写 mode,Mission Manager 写 phase 内部状态";具体每条 phase 转换的 owner 由 D3 + D4 + D5 协同确认
  - 处置:open;由 D3 / D4 协同(Wave 8 co-seal)锁定;若发现拆分不合理,本文件 §8 变更日志记录 + 退化为 "phase 推进归 Mode Manager,Mission 仅退缩到 wp 序列管理"
- **D5 leaf 注入的具体 PARAM 字段未列(D5 owns)**
  - 影响:§4.7 列出 4 处 leaf 注入点(Shaper profile / Source stick / Mission default / Mode guard);但每个注入点对应的 B3 PARAM 字段子集未枚举(D5 范围)
  - 处置:由 [D5](D5-multicopter-leaf.md)(Wave 9)枚举具体 PARAM 字段名 + 数值默认值
- **电池电压字段来源未定(D1 §5 已 open)**
  - 影响:§4.3.3 Safety Monitor 输入项标 "电池电压字段 TBD";Phase 2 闭环切片若不含电量场景,可暂搁
  - 处置:延续 D1 §5 同条 open;由 [H4](../H-verification/H4-fault-catalog.md) 决定接口

## 6. 退出条件复核

对照 [`00-design-plan.md §4.D`](../00-design-plan.md) D2 退出条件:

> **D2 退出条件原文**:6 大子模块(Source Selector / Mode Mgr / Safety / Mission / Shaper / Output Assembler)的边界与内部信号

| # | 退出条件原文(分项) | 本文档依据 | 状态 |
|---|---|---|---|
| 1 | Source Selector 子模块边界 | §4.1.2 (1) + §4.3.1(输入/输出/Reset/速率/Owner/Stateflow=否) | 满足 |
| 2 | Mode Manager 子模块边界 | §4.1.2 (2) + §4.3.2(输入/输出/Reset/速率/单一 Stateflow chart;chart 内部 D3 owns) | 满足 |
| 3 | Safety / Failsafe Monitor 子模块边界 | §4.1.2 (3) + §4.3.3(输入/输出/Reset/速率/Owner) | 满足 |
| 4 | Mission / Auto Manager 子模块边界 | §4.1.2 (4) + §4.3.4(输入/输出/Reset/速率/Owner) | 满足 |
| 5 | Command Shaper 子模块边界 | §4.1.2 (5) + §4.3.5(输入/输出/Reset/速率/Owner;内部 D4 owns 公式) | 满足 |
| 6 | Output Assembler 子模块边界 | §4.1.2 (6) + §4.3.6 + §4.6(输入/输出/Reset/速率/字段路由四类) | 满足 |
| 7 | 6 子模块之间的内部信号 | §4.1.1 顶层框图 + §4.2.1 内部 bus 清单(`FMS_SrcSel_Bus` / `FMS_SrcHealth_Bus` / `FMS_Safety_to_ModeMgr_Bus` / `FMS_Mission_to_ModeMgr_Bus` / `FMS_Mission_to_Shape_Bus` / `FMS_ModeMgr_to_Shape_Bus` / `FMS_ModeMgr_to_Asm_Bus` / `FMS_Shape_to_Asm_Bus` / `FMS_Passthrough_Direct_Bus`) | 满足 |

## 7. 下游影响

按 [`01-design-relationships.md §4.4`](../01-design-relationships.md) D2 出边 + §4.10 A8 ⇢ D2 反向:

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [D3 Mode Manager](D3-mode-manager.md)(Wave 8) | → | Mode Manager 子模块边界(§4.3.2)+ Stateflow 输入/输出 bus 字段类别(§4.2.1)+ reset entry state(§4.8.1)+ §4.4 hand-off 表 |
| [D4 Command Shaper](D4-command-shaper.md)(Wave 8) | → | Shaper 子模块边界(§4.3.5)+ `FMS_Shape_to_Asm_Bus` 字段类别(§4.2.1)+ routing 指令接口(§4.2.1 `FMS_ModeMgr_to_Shape_Bus`)+ §4.5 hand-off 表 |
| [D5 多旋翼 leaf](D5-multicopter-leaf.md)(Wave 9) | →(经 D3/D4) | §4.7 — 6 子模块 全 shared;leaf 注入 4 处(PARAM-driven) |
| [D6 FMS↔Controller 接口](D6-fms-controller-interface.md)(Wave 8) | ↔ | §4.6.2 — cmd_mask 单一写者(Shaper)、Output Assembler 不修改、reset = 全 0 |
| [E1 Controller 功能](../E-controller/E1-controller-functional.md)(Wave 8) | ⇢(经 `FMS_Out_Bus`) | §4.2.1 `FMS_Shape_to_Asm_Bus` 字段类别 → `FMS_Out_Bus` 消费侧 |
| [F3 INS 消费规则](../F-ins-contract/F3-consumption-rules.md)(Wave 9) | ⇢ | §4.3.2 / §4.3.3 / §4.3.5 — `INS_Out_Bus` 在 Mode Manager / Safety / Shaper 三处的消费点 |
| [G1 MIL 顶层](../G-harness/G1-mil-toplevel.md)(Wave 10) | ⇢ | §4.1.1 顶层框图(FMS 作为单一 model reference);§4.7 6 子模块全 shared(macro 不切分到顶层) |
| [A8 共享库块](../A-architecture/A8-shared-library-roster.md)(逆向 ⇢ A8 ⇢ D2,per [01 §4.10](../01-design-relationships.md)) | ⇢ | 本文件 §4.3.x "A8 shell" 行复核 [A8 §4.3.2](../A-architecture/A8-shared-library-roster.md) 7 个 FMS shell 全部被认领(`FMS_Source_Selector_Shell` / `FMS_Mode_Manager_Framework` / `FMS_Safety_Monitor_Framework` / `FMS_Mission_Decoder_Shell` / `FMS_Command_Shaper_Shell` / `FMS_Output_Assembler_Shell` 各对应一个子模块;`FMS_Reset_Hooks_Shell` 横切注入 §4.8) |

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-08 | Wave 7 D2 author | 初稿 — 6 子模块拓扑(D1 §4.8 macro 保留)+ 9 内部 bus + D3/D4/D5/D6 hand-off + Output Assembler 字段四路由 + A6 init/reset 协议 |

## Self-check

- [x] frontmatter 完整,字段值合法
- [x] 上游文档全部存在且 status ≥ reviewed,**或**为本工作项所在 co-seal batch 的同批兄弟(D1 = reviewed 2026-05-08;A1..A8 / B1..B3 / 架构 v1 全 reviewed 或 accepted;无同 batch 兄弟引用)
- [x] 退出条件逐条复核完成,每条均给出依据(§6 表;7 行 = 6 子模块 + 内部信号)
- [x] 引用路径全部可点击访问(架构 v1 / A1..A8 / B1..B3 / D1 / D3..D6 / E1 / F3 / G1 / G2 / H4 均使用相对路径)
- [x] 不存在 RULES §5 禁则中的内容(无 .slx 截图;无 .m 可执行代码;无 firmware 实现细节复述;无 PR/branch 名;无 bus/enum 字段表重复定义 — 字段类别引用 A3 / B1 / B2 / B3 而不重列)
- [N/A] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记 — `contract_impact: no`,N/A
- [N/A] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径 — 本文件不镜像 firmware 契约,所有契约引用走 B1/B2/B3;firmware reference 已记入 §3 依赖表为 `<pending hash>` placeholder(per A3/B1 batch 约定)
- [x] 下游影响已沿关系图识别完毕(§7 表覆盖 [01 §4.4](../01-design-relationships.md) D2 出边 D3/D4 + 衍生 D5/D6/E1/F3/G1 + §4.10 A8 ⇢ D2 反向)
- [x] 文档不超出本工作项范围(无越权设计 — 不锁 Stateflow 内部 / 不锁整形公式 / 不锁多旋翼 leaf 数值 / 不锁 cmd_mask 位语义)
