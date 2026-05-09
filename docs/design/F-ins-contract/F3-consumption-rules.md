---
work_item: F3
title: INS_Out_Bus 消费规则(模型仓侧字段依赖矩阵 / validity 处理 / fallback)
upstream: ["架构 v1", "A1", "A3", "A6", "A7", "B1", "B2", "B5", "D1", "D3", "D4", "D6", "E1", "F1", "F2"]
contract_impact: no
status: draft
authored_at: 2026-05-09
last_reviewed_at:
reviewer_verdict: none
---

# F3 INS_Out_Bus 消费规则(模型仓侧字段依赖矩阵 / validity 处理 / fallback)

## 1. 目的

为模型仓侧的全部 `INS_Out_Bus` 消费方(FMS 6 子模块 + Controller L-01..L-04 + ins_stub passthrough + harness/log)在**字段层**和**validity-flag 层**给出一份**单一来源**的消费规则册:每个字段由谁消费;每个 validity 位在置 0 时各消费方应执行的 fallback 行为;recovery 时的 holdoff 与 bumpless 政策;timestamp staleness 检测策略;以及边界场景(冷启动、瞬时 glitch、持续失效、多位同时翻转)的裁决。本文件把 [A3 §4.8](../A-architecture/A3-module-boundaries.md) INS 消费侧依赖矩阵从"标记表"升级为"可执行规则",并把 [D1 §4.4](../D-fms/D1-fms-functional.md) / [D6 §4.5](../D-fms/D6-fms-controller-interface.md) / [E1 §4.5](../E-controller/E1-controller-functional.md) 散落在三个文档中的 INS 相关条款集中复述并对齐。

## 2. 范围

**在范围:**

- §4.1 **per-field 消费方矩阵**:每个 `INS_Out_Bus` 字段在 FMS / Controller / ins_stub passthrough / harness log 四列下的消费方枚举(echo + 扩展 [A3 §4.8](../A-architecture/A3-module-boundaries.md))
- §4.2 **per-consumer validity 处理规则**:FMS(D1 TR-03/TR-04 + D3 mode entry guards + arm guard)与 Controller(L-01..L-04 per-loop gating)在每条 validity-flag 上的应有行为
- §4.3 **fallback 政策表**:每条 validity-drop 情境下 FMS 与 Controller 的 fallback 动作 + 恢复条件
- §4.4 **recovery / re-engagement holdoff**:验证位由 0 → 1 时的 holdoff 时长与 bumpless 政策
- §4.5 **timestamp staleness 检测**:FMS Source Selector 用 `INS_Out_Bus.timestamp` 做 staleness gate 的判据 + Controller 不消费 timestamp 的语义
- §4.6 **per-subsystem 规则速查**:把 §4.1..§4.5 按消费方汇总以便引用
- §4.7 **边界场景**:冷启动 / 瞬时 glitch / 持续失效 / 多位同时翻转 / 致命位与降级位优先级
- §4.8 **cross-references**:与 A3 / D1 / D6 / E1 / F2 / B5 / H4 的指针对照

**不在范围(由其他工作项处理):**

- `INS_Out_Bus` schema(字段顺序 / 字节布局 / 类型) — 由 [B1 §4.3.1](../B-contracts/B1-bus-inventory.md) + [B5](../B-contracts/B5-ins-bus-mirror.md) mirror artifact 锁定;F3 只引用,不复述
- `INS_Status` / `INS_Flag` 数值 / 位号 — 由 [B2 §4.4.10 / §4.4.11](../B-contracts/B2-enum-inventory.md) 锁定
- B5 mirror PR 工作流(`canonical 来源` / 抽取脚本 / 版本标记) — 由 [B5](../B-contracts/B5-ins-bus-mirror.md) + [I2](../I-tooling/I2-bus-enum-mirror.md) 处理;F3 仅声明:firmware 改字段 → B5 镜像 → F3 fallback 表必须重新校验
- D1 失效触发 ID / 阈值 / 仲裁优先级(TR-01..TR-11 全集 / failsafe ladder) — 由 [D1 §4.4](../D-fms/D1-fms-functional.md) 锁定;F3 仅 echo + 扩展 INS 相关项(TR-03 / TR-04)
- Controller 各环数学公式(P / PI / cascade gain) — 由 [E3](../E-controller/E3-loops-algorithm.md) 处理;F3 只锁定 "validity-drop 时的功能意图"(zero-frozen / hold-last / 紧急 hold)
- D6 cmd_mask 互斥规则 / MX-1..MX-10 — 由 [D6 §4.4 / §4.5](../D-fms/D6-fms-controller-interface.md) 锁定;F3 仅引用 INS-invalid 触发的 emission policy
- INS_Status enum 的 init / firmware 内部 transition — firmware-owned;F3 只规定**消费侧**对 enum 各值的 gate 政策
- 故障注入时 INS validity 的具体波形 / 触发条件 — 由 [H4 故障注入目录](../H-verification/H4-fault-catalog.md) 处理(Wave 11);F3 仅声明 fallback 表必须覆盖 H4 注入产生的所有 validity 失效情境

## 3. 依赖

| 上游 | 引用位置 | 用途 |
|---|---|---|
| 架构 v1 §3 / §5.1.6 / §12.3 / §13 / §17 risk 2 | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | INS firmware-owned(§3 / §12.3);§5.1.6 INS validity 必须在 FMS / Controller 边界干净消费;§13 INS row 10 ms cadence;§17 risk 2 INS_Out_Bus 契约漂移风险 — F3 fallback 表是契约漂移的"消费侧防线" |
| [A1 架构 v1 评审与缺口闭合](../A-architecture/A1-v1-review.md) (status: reviewed) | A1 §5.1.6 把"Controller Inputs/Outputs 缺字段方向矩阵"分配给 A3 + F3;A3 已 reviewed,F3 在该框架内为消费规则落地 | F3 是 A1 §5.1.6 缺口的最终闭合(消费规则层) |
| [A3 模块边界信号清单细化](../A-architecture/A3-module-boundaries.md) (status: reviewed, 2026-05-07) | §4.4.4 FMS 输入 `INS_Out_Bus` 字段表 + 子模块依赖;§4.5.3 Controller 输入 `INS_Out_Bus` 字段表 + 子模块依赖;§4.8 INS 消费侧依赖矩阵集中表 | **A3 §4.8 是 F3 §4.1 的权威源**;F3 §4.1 是该矩阵的"可执行扩展"(echo + 加 ins_stub passthrough 列 + harness log 列) |
| [A6 Init/Reset 状态契约](../A-architecture/A6-init-reset-contract.md) (status: reviewed) | §4.5.3 INS 协调协议(模型仓侧消费者 init 后必须先 gate `INS_Status.ready` / `INS_Flag` 才离开 disarmed)| F3 §4.2 / §4.7 cold-start 行为来源 |
| [A7 时间与时间戳约定](../A-architecture/A7-time-conventions.md) (status: reviewed) | §4.1 timestamp 单位 ms / §4.2 epoch zero-from-boot / §4.5 Controller 不基于 timestamp 派生 dt | F3 §4.5 staleness 检测的语义基础 |
| [B1 Bus 清单与 schema 设计](../B-contracts/B1-bus-inventory.md) (status: reviewed) | §4.3.1 `INS_Out_Bus` 10 字段表 + §4.6.5 / §4.6.6 `INS_Status` / `INS_Flag` 子 schema | F3 §4.1 字段集 = B1 §4.3.1;F3 不复述 schema |
| [B2 Enum 清单与数值锁定](../B-contracts/B2-enum-inventory.md) (status: reviewed) | §4.4.10 `INS_Status` 值表(NOT_READY / INITIALIZING / READY / DEGRADED / FAULT);§4.4.11 `INS_Flag` 位号(POSITION_VALID / VELOCITY_VALID / ATTITUDE_VALID / HEADING_VALID / MAG_VALID / GPS_VALID / BARO_VALID / AIRSPEED_VALID / VEL_BODY_VALID)| F3 fallback 表上的位名直接引用 B2,不复述数值 |
| [B5 INS_Out_Bus 镜像同步流程](../B-contracts/B5-ins-bus-mirror.md) (status: reviewed, 2026-05-07, contract_impact=yes) | §4.5 mirror artifact 存放;§4.8 F-area 下游耦合;镜像 schema 任何变更必须由 F3 fallback 表重新校验 | F3 §2 / §4.8 / §5 镜像变更触发 F3 重新评审的纪律来源 |
| [D1 FMS 功能设计](../D-fms/D1-fms-functional.md) (status: reviewed, 2026-05-08) | §4.4.1 触发清单 TR-03(INS validity loss)+ TR-04(GPS lost);§4.4.2 触发优先级(TR-03 与 TR-10 同最高级);§4.4.3 debounce / hysteresis(`ins_unready_timeout_s` × 50 Hz);§4.4.4 可恢复性 | F3 §4.2 FMS 规则 + §4.3 fallback 表 FMS 列直接 echo D1 |
| [D3 FMS Mode Manager 详细设计](../D-fms/D3-mode-manager.md) (Wave 8 co-seal sibling) | mode-entry guard:POSHOLD / AUTO 要求 `position_valid && velocity_valid`;arm guard:`INS_Status >= READY && attitude_valid` | F3 §4.2 FMS 规则中的 mode entry / arm guard 条目;**D3 在同 Wave 8 同 batch reviewed,F3 是其下游(Wave 9),不存在 forward-cite 风险** |
| [D6 FMS↔Controller 接口约定](../D-fms/D6-fms-controller-interface.md) (status: reviewed, 2026-05-08) | §4.5 INS-invalid 情境下 FMS 应 emit 的 cmd_mask 政策(完全失效=cmd_mask=0;位置失效=不 emit BIT_POS;速度失效=不 emit BIT_POS/VEL/ACC;姿态失效=cmd_mask=0;偏航失效=不 emit BIT_YAW;GPS 失效=由 INS_Flag 仲裁) | F3 §4.2 / §4.3 直接 echo D6 §4.5;F3 锁 Controller 的对应行为(cmd_mask=0 → MX-1 disarmed-equiv;BIT_POS=1 但 position_valid=0 → MX-3 拒绝 + 降级)|
| [E1 Controller 功能设计](../E-controller/E1-controller-functional.md) (status: reviewed, 2026-05-08) | §4.5.4 / §4.5.5 / §4.5.6 各环 measurement gate(L-01:position_valid;L-02:velocity_valid;L-03/L-04:attitude_valid 致命);§4.4.2 zero-frozen vs hold-last;§4.4.4 cmd_mask 切换 integrator 处置 | F3 §4.2 Controller 规则 + §4.3 fallback 表 Controller 列直接 echo E1 |
| [F1 ins_stub 功能设计](F1-ins-stub-functional.md) (status: reviewed, 2026-05-08) | §4.3.5 transient timeline(NOT_READY → INITIALIZING → READY 各 valid 位时序)| F3 §4.7 cold-start 行为引用 F1 timeline 作为 ins_stub 侧 supply 模型 |
| [F2 ins_stub 结构设计](F2-ins-stub-structural.md) (status: reviewed, 2026-05-08) | §4.4 init transient 三态 chart;§4.3 字段映射表(INS_Out_Bus 字段从 Plant_States_Bus 派生)| F3 §4.1 ins_stub passthrough 列直接引用 F2 §4.3 字段表 |
| firmware INS 头文件 | `FMT-Firmware/src/model/ins/<variant>/lib/INS_types.h` @ `FMT-Firmware @ <pending hash>`(commit hash 在 INDEX 决策日志登记本工作项时补齐,与 F-area 同 batch 占位约定;FMT-Firmware 仓未挂载在本环境)| INS_Out_Bus / INS_Status / INS_Flag 的 firmware-canonical schema 真理来源,经 [B5](../B-contracts/B5-ins-bus-mirror.md) mirror artifact 间接到达;F3 **不**直接读 firmware 头 |

注:本工作项 contract_impact=**no**。F3 只规定**消费侧规则**,不修改 `INS_Out_Bus` 任一字段 / `INS_Status` / `INS_Flag` 任一位的契约语义。`INS_Out_Bus` schema 由 [B1](../B-contracts/B1-bus-inventory.md) + [B5](../B-contracts/B5-ins-bus-mirror.md) 锁定(那两个 contract_impact=yes);`INS_Status` / `INS_Flag` enum 由 [B2](../B-contracts/B2-enum-inventory.md) 锁定。F3 对 firmware 完全透明。

## 4. 设计内容

### 4.1 Per-field 消费方矩阵(echo + 扩展 [A3 §4.8](../A-architecture/A3-module-boundaries.md))

下表把 [A3 §4.8](../A-architecture/A3-module-boundaries.md) 的 FMS 列 + Controller 列**收敛为消费方枚举**,并新增 ins_stub passthrough 列(producer-side 语义)与 harness/log 列。每行字段名按 [B1 §4.3.1](../B-contracts/B1-bus-inventory.md) / [B5](../B-contracts/B5-ins-bus-mirror.md) mirror artifact 命名;若 firmware tip 字段名漂移,以 mirror artifact 为权威。

| `INS_Out_Bus` 字段 | FMS 消费方 | Controller 消费方 | ins_stub passthrough(F2 §4.3) | harness / log |
|---|---|---|---|---|
| `position_lla` (lat_deg / lon_deg / alt_m) | Mission/Auto Manager(航点 LLA 算术 + home arming 判定);Output Assembler(可选 echo 到 GCS-facing log)| (无直接消费 — Controller 用 NED) | F2 块 2:从 `pos_ned_m` + `home_lla` 派生(NED→LLA WGS-84 平面近似)| 必记;G3 logsout |
| `position_ned_m[3]` | Mission/Auto Manager(geofence breach 检测 = TR-07);Mode Manager(POSHOLD/MISSION entry guard);Source Selector(可用性枚举)| L-01 Position Loop(measurement)| F2 块 2:identity from `Plant_States_Bus.pos_ned_m` | 必记 |
| `velocity_ned_mps[3]` | Command Shaper(stick→vel 整形闭环参考);Source Selector(降级策略)| L-02 Velocity Loop(measurement)| F2 块 2:identity from `Plant_States_Bus.vel_ned_mps` | 必记 |
| `quat_ned_to_b[4]` | Mode Manager(arm guard 姿态自检);Command Shaper(frame transform — stick→body)| L-03 Attitude Loop(measurement)| F2 块 2:identity from `Plant_States_Bus.quat_ned_to_b` + post-noise normalize | 必记 |
| `euler_ned_to_b_rad[3]` | (仅 log;quat 是消费规范字段)| (仅 log;若 B1 锁 euler 替代 quat 路径,L-03 二选一)| F2 块 2:从 quat 派生(quat→euler 转换)| 必记 |
| `ang_rate_b_radps[3]` | (无 — FMS 不控制 rate)| L-04 Rate Loop(measurement,filtered);L-03(rate FF,可选)| F2 块 2:identity from `Plant_States_Bus.ang_rate_b_radps` | 必记 |
| `acc_b_mps2[3]` | (无)| L-02(可选 acceleration FF;Phase 2 默认不开,per [E1 §4.1.1](../E-controller/E1-controller-functional.md) 不变量 3)| F2 块 2:identity from `Plant_States_Bus.acc_b_mps2` | 必记 |
| `INS_Status`(enum,per B2 §4.4.10)| Mode Manager(arm guard:`INS_Status >= READY` 才允许 arm);Safety Monitor(TR-03 触发器:`INS_Status != READY` 持续 N 帧)| (无直接消费;Controller 仅看 `flag.attitude_valid` + cmd_mask)| F2 块 5(Init Transient chart)输出 | 必记 |
| `INS_Flag.INS_FLAG_BIT_ATTITUDE_VALID` | Mode Manager(M-04 ALTHOLD / M-05 POSHOLD entry guard;TR-03 触发);Command Shaper(frame transform gate)| L-03 Attitude / L-04 Rate gate(critical;失效→零推力 5 ms)| F2 块 5 + 块 6 health gate | 必记 |
| `INS_Flag.INS_FLAG_BIT_POSITION_VALID` | Mode Manager(M-05/M-08/M-09/M-10 entry guard);Mission/Auto Manager(autoarm 条件 + waypoint 算术安全);Safety Monitor(TR-04 触发)| L-01 Position gate | F2 块 5 + 块 6 | 必记 |
| `INS_Flag.INS_FLAG_BIT_VELOCITY_VALID` | (FMS 无直接 gate — D6 §4.5 把"位置失效"自动级联到 vel/acc 不 emit)| L-02 Velocity gate;cascade 降级到 attitude-only | F2 块 5 + 块 6 | 必记 |
| `INS_Flag.INS_FLAG_BIT_HEADING_VALID` | (FMS 无直接 gate — 由 mode 表暗含,POSHOLD/AUTO 间接要求)| L-03 yaw 子轴 gate(yaw 角参考闭环);L-04 yaw rate 不 gate(yaw rate 是 raw rate,仍可)| F2 块 5 + 块 6 | 必记 |
| `INS_Flag.INS_FLAG_BIT_GPS_VALID` | Source Selector(mission availability 降级);Safety Monitor(TR-04 GPS lost)| (无直接消费 — GPS 失效但 position_valid 仍 1 时 Controller 不退化,per D6 §4.5 GPS 单独失效行)| F2 块 5 + 块 6 | 必记 |
| `INS_Flag.INS_FLAG_BIT_MAG_VALID` | Safety Monitor(degraded warning;TR-XX 不直接触发)| (无)| F2 块 5 + 块 6 | 必记 |
| `INS_Flag.INS_FLAG_BIT_BARO_VALID` | Safety Monitor(altitude staleness)| (无 — Controller 不直接消费 baro;altitude 由 NED z 通道得来)| F2 块 5 + 块 6 | 必记 |
| `INS_Flag.INS_FLAG_BIT_AIRSPEED_VALID` | (固定翼 / VTOL — Phase 2 多旋翼 reserved)| (固定翼 — Phase 2 reserved)| F2 块 5 + 块 6 | 必记 |
| `INS_Flag.INS_FLAG_BIT_VEL_BODY_VALID`(若 mirror 含此位)| (无直接 gate)| (若 E3 决议引入 body-velocity 路径)| F2 块 5 | 必记 |
| `timestamp` (uint32 ms, A7 §4.1)| Source Selector(staleness gate,经 `Stale_Detector` per [A8 §4.2.5](../A-architecture/A8-shared-library-roster.md))| (不消费 — Controller 用固定步长派生 dt,per [A7 §4.5](../A-architecture/A7-time-conventions.md))| F2 块 7:harness clock echo | 必记 |

记号 / 注:

- **"Mission/Auto Manager"**:对应 [D2](../D-fms/D2-fms-structural.md) Mission 子模块;[D1 §4.1](../D-fms/D1-fms-functional.md) M-09/M-10 等 mode。
- **"Mode Manager"**:对应 D3。
- **"Command Shaper"**:对应 [D4](../D-fms/D4-command-shaper.md)。
- **"Source Selector"** / **"Safety Monitor"** / **"Output Assembler"**:对应 D2 子模块。
- **L-01..L-04**:对应 [E1 §4.1.2](../E-controller/E1-controller-functional.md) 4 个环路。
- "F2 块 N" 引用 [F2 §4.7](F2-ins-stub-structural.md) 7 个块的编号。
- 本表是 [A3 §4.8](../A-architecture/A3-module-boundaries.md) 的"可执行投影":A3 列分子模块,F3 列分消费方 + 加 ins_stub passthrough(producer 侧)+ harness log(observer 侧),覆盖 INS 字段在模型仓的全消费链。

### 4.2 Per-consumer validity 处理规则

#### 4.2.1 FMS 规则(echo + 对齐 [D1 §4.4](../D-fms/D1-fms-functional.md) + [D6 §4.5](../D-fms/D6-fms-controller-interface.md) + [D3](../D-fms/D3-mode-manager.md))

| 规则 ID | 触发字段 / 条件 | 消费方 | 应有行为 | 来源 |
|---|---|---|---|---|
| F3-FMS-01 | `INS_Status < INS_STATUS_READY`(B2 §4.4.10)持续 ≥ N=`ins_unready_timeout_s` × 50 帧 | Safety Monitor / Mode Manager | 触发 D1 TR-03 INS validity loss → 若 INAIR:先切 M-07 LAND,再切 M-01 LOCKDOWN(FS-04);若 ON-GROUND:直接切 M-01 LOCKDOWN(FS-04);`failsafe_state=FAIL_INS_INVALID`;`error_code=ERR_INS_INVALID`;Command Shaper emit `cmd_mask=0`(per D6 §4.5 完全失效行)| D1 §4.4.1 TR-03;D6 §4.5;A6 §4.5.3 |
| F3-FMS-02 | `INS_FLAG_BIT_ATTITUDE_VALID == 0` 持续 ≥ N 帧 | Safety Monitor / Mode Manager | 触发 D1 TR-03 INS validity loss(attitude 子情况 — critical);行为同 F3-FMS-01;额外:Command Shaper emit `cmd_mask=0`(per D6 §4.5 姿态失效行,FMS 触发 emergency)| D1 §4.4.1 TR-03;D6 §4.5 |
| F3-FMS-03 | `INS_FLAG_BIT_POSITION_VALID == 0` 持续 ≥ N 帧,且 attitude / velocity 仍有效 | Mode Manager / Command Shaper | 若当前 mode ∈ {M-05 POSHOLD, M-08 RTL, M-09 TAKEOFF/LAND, M-10 MISSION}:降级到 M-04 ALTHOLD(若高度环可用 — `velocity_valid==1` 时垂向闭环仍可用);否则降级到 M-03 STABILIZE;`failsafe_state=FAIL_INS_INVALID`(子情况);`error_code=ERR_GPS_LOST`(若 GPS 也失效)或 ERR_INS_INVALID;Command Shaper:**不**emit `MASK_BIT_POSITION_LOOP`,允许 BIT_VEL / BIT_ATT / BIT_RATE / BIT_YAW / BIT_YAW_RATE / BIT_THROTTLE_PASSTHROUGH | D1 §4.4.1 TR-04 范围扩展;D6 §4.5 位置失效行 |
| F3-FMS-04 | `INS_FLAG_BIT_VELOCITY_VALID == 0` 持续 ≥ N 帧,且 attitude 仍有效 | Mode Manager / Command Shaper | 降级到 M-03 STABILIZE(垂向速度环也不再可用);Command Shaper:**不**emit BIT_POS / BIT_VEL / BIT_ACC;允许 BIT_ATT / BIT_RATE / BIT_YAW / BIT_YAW_RATE / BIT_THROTTLE_PASSTHROUGH | D6 §4.5 速度失效行 |
| F3-FMS-05 | `INS_FLAG_BIT_HEADING_VALID == 0` 持续 ≥ N 帧,且 attitude / velocity / position 仍有效 | Command Shaper | **不**emit `MASK_BIT_YAW_LOOP`(yaw 角环);可继续 emit `MASK_BIT_YAW_RATE_LOOP`(纯 rate,不需绝对 heading);允许 BIT_ATT(roll/pitch only)| D6 §4.5 偏航失效行 |
| F3-FMS-06 | `INS_FLAG_BIT_GPS_VALID == 0` 持续 ≥ N 帧,但 position_valid / velocity_valid 仍 1(光流 / 视觉源 backing)| Source Selector / Safety Monitor | 触发 D1 TR-04 GPS lost(子情况);Source Selector 把 mission 标记为不可启动;**不**强制降级 mode(per D6 §4.5 GPS 单独失效行 — 由 INS_Flag 仲裁,而非直接由 GPS_VALID gate)| D1 §4.4.1 TR-04;D6 §4.5 GPS 行 |
| F3-FMS-07(**arm guard**)| `INS_Status < INS_STATUS_READY` **或** `INS_FLAG_BIT_ATTITUDE_VALID == 0`(无 N 帧 debounce — arm 是显式 user 触发,即时 gate)| Mode Manager(D3 arm 入口)| 拒绝 arm 请求;停留 M-01 DISARMED;`error_code=ERR_INS_INVALID`(per D1 §4.4 TR-08 也覆盖)| D1 §4.1.2 M-01 离开条件;A6 §4.5.3 第 2 项 |
| F3-FMS-08(**mode entry guard**)| 进入 M-04 ALTHOLD 请求 | Mode Manager(D3 transition guard)| 仅当 `INS_FLAG_BIT_ATTITUDE_VALID == 1` 才允许进入(per D1 §4.1.1 mode 表前提列);否则拒绝转换,`error_code=ERR_MODE_INVALID`(D1 §4.4.1 TR-08)| D1 §4.1.1 M-04 行 |
| F3-FMS-09(**mode entry guard**)| 进入 M-05 POSHOLD / M-08 RTL / M-09 TAKEOFF/LAND / M-10 MISSION 请求 | Mode Manager(D3 transition guard)| 仅当 `INS_FLAG_BIT_POSITION_VALID == 1` **且** `INS_FLAG_BIT_VELOCITY_VALID == 1` 才允许进入;否则拒绝(D1 §4.4.1 TR-08)| D1 §4.1.1 M-05 / M-08 / M-09 / M-10 行 |
| F3-FMS-10(**timestamp staleness**)| `now_ms - INS_Out_Bus.timestamp > T_stale`(默认 50 ms,§4.5)| Source Selector(经 [A8 §4.2.5](../A-architecture/A8-shared-library-roster.md) Stale_Detector)| 视为 `INS_Status < READY` 等价 → 走 F3-FMS-01 路径(完全失效)| D1 §4.4.1 TR-03 staleness 子情况;A7 §4.4 |

debounce 数值(N)= `ins_unready_timeout_s` × 50 Hz(D1 §4.4.3 锁,FMS_PARAM.24)。**arm guard(F3-FMS-07)无 debounce**(arm 是 user 显式触发,瞬时拒绝)。

#### 4.2.2 Controller 规则(echo + 对齐 [E1 §4.5](../E-controller/E1-controller-functional.md))

| 规则 ID | 触发字段 / 条件 | 消费方 | 应有行为 | 来源 |
|---|---|---|---|---|
| F3-CTL-01(**L-01 Position**)| `INS_FLAG_BIT_POSITION_VALID == 0`(单帧即生效;Controller 5 ms 周期无 debounce — debounce 在 FMS 50 Hz 层)| L-01 Position Loop | L-01 旁通(等价 [E1 §4.4.1](../E-controller/E1-controller-functional.md) bypass 语义);integrator freeze 然后清零(per [E1 §4.4.4](../E-controller/E1-controller-functional.md) reset 策略);`vel_sp_inner` 输出 = 0(zero-frozen,默认策略,per [E1 §4.4.2](../E-controller/E1-controller-functional.md));L-02 改用 `use_external` 路径(若 cmd_mask `BIT_VEL=1`)或继续 `use_inner` 但消费 0 setpoint | E1 §4.5.4 |
| F3-CTL-02(**L-02 Velocity**)| `INS_FLAG_BIT_VELOCITY_VALID == 0`(单帧)| L-02 Velocity Loop | L-02 旁通;integrator freeze + 清零;输出虚拟控制 = 0 → cascade 降级:L-03 改用上游 mode 退化(D1 TR-04)或 hold-last;Controller 不自行决定 mode 退化(per E1 §4.2.1 C-10 行为)| E1 §4.5.5(类比);D6 §4.5 速度失效行 |
| F3-CTL-03(**L-03 Attitude — CRITICAL**)| `INS_FLAG_BIT_ATTITUDE_VALID == 0`(单帧)| L-03 Attitude Loop | **CRITICAL 失效**:L-03 不产生有效 `ang_rate_sp_inner`;触发 D1 TR-03 INS 失效联动(per E1 §4.5.6 critical 注);Controller 立即(同帧,5 ms 周期 — 比 FMS 20 ms TR-03 debounce 快至少 4 帧)输出**零推力 + 角速率参考 0 + 全 integrator 清零**(等价 [E1 §4.2.3](../E-controller/E1-controller-functional.md) C-10 illegal fallback 安全态,与 [A6 §4.4.2](../A-architecture/A6-init-reset-contract.md) Controller 输出零推力稳态一致)| E1 §4.5.6;A6 §4.4.2 |
| F3-CTL-04(**L-04 Rate — CRITICAL**)| `INS_FLAG_BIT_ATTITUDE_VALID == 0`(单帧;**rate 环 gate 复用 attitude_valid**,per [A3 §4.5.3](../A-architecture/A3-module-boundaries.md) 注:Phase 2 默认不开 raw IMU 路径)| L-04 Rate Loop | 同 F3-CTL-03 行为(姿态环 + 速率环联动 critical 失效;Controller 输出零推力)| E1 §4.5.7;A3 §4.5.3 |
| F3-CTL-05(**L-03 yaw 子轴**)| `INS_FLAG_BIT_HEADING_VALID == 0` 但 `INS_FLAG_BIT_ATTITUDE_VALID == 1` | L-03 yaw 子轴 | yaw 角参考 hold(用最后一帧有效 yaw,或退化到 mag-only 估计 — 由 [E3](../E-controller/E3-loops-algorithm.md) 锁具体算法);roll/pitch 子轴正常运行;若 cmd_mask `BIT_YAW_RATE=1`,继续走 yaw rate 通道(纯 rate,不需 heading)| E1 §4.5.6 yaw 子通道;D6 §4.5 偏航失效行 |
| F3-CTL-06(**MX-3 拒绝**)| FMS emit `cmd_mask.MASK_BIT_POSITION_LOOP=1` 但 `INS_FLAG_BIT_POSITION_VALID == 0`(违反 D6 §4.5 良民规则)| Mask Resolver([E2 §4.1.1](../E-controller/E2-controller-structural.md))| 把 BIT_POS 视为清位,降级到 BIT_VEL / BIT_ATT 等;发 `ERR_INS_INVALID` 警告(per D6 §4.4 MX-3)| D6 §4.4 MX-3;E1 §4.2 |
| F3-CTL-07(**INS_Status 无直接消费**)| `INS_Status < INS_STATUS_READY` | (Controller 不直接消费;通过 cmd_mask 间接传达,per A6 §4.5.3 第 3 项)| 当 FMS 因 INS_Status 失效而 emit cmd_mask=0 → Controller 见 cmd_mask=0 → 走 D6 MX-1 disarmed-equiv(零推力 + integrator 清零);Controller 不自行解释 INS_Status enum | A6 §4.5.3;D6 §4.4 MX-1 |
| F3-CTL-08(**timestamp 不消费**)| (任意 timestamp 值)| (无 — Controller 用固定步长 5 ms 派生 dt,per A7 §4.5)| 无行为;timestamp 仅由 harness Stale_Detector(若启用)或 G3 logsout 用 | A7 §4.5;A3 §4.5.3 注 |

**FMS-vs-Controller 时间常数对比**:

- FMS 触发 TR-03 / TR-04 的 **debounce 时长** ≈ `ins_unready_timeout_s` 数百 ms 量级(防瞬时抖动)。
- Controller per-loop gate **无 debounce**,5 ms 周期内即生效。
- 关键含义:**姿态失效时,Controller 的 5 ms 紧急零推力比 FMS 的 ~20 ms TR-03 路径更早发出**;FMS 路径用于产生 mode + cmd_mask + failsafe_state 跨 50 Hz 决策;Controller 路径用于即时 5 ms 安全态。两条路径互不依赖,形成"快慢两线" 的纵深防御。
- 这条不变量是 §4.7 边界场景"多位同时翻转"裁决的物理基础。

### 4.3 Fallback 政策表(per validity-flag drop scenario)

下表是 §4.2 的"按情境维度"重排,作为代码实现 / 测试 oracle 的权威 cross-table。每行代表一个 validity 失效情境,列出 FMS 与 Controller 的应有动作 + 恢复条件。

| Drop 情境 | FMS 行为 | Controller 行为 | 恢复条件 |
|---|---|---|---|
| **`INS_Status < INS_STATUS_READY`**(冷启动 / firmware-reported 完全失效;持续 ≥ N 帧)| F3-FMS-01:TR-03 → 若 INAIR M-07 LAND 后 M-01 LOCKDOWN;否则直接 M-01 LOCKDOWN(FS-04);`cmd_mask=0` | F3-CTL-07 / D6 MX-1:Controller 见 cmd_mask=0 → 零推力 + 全 integrator 清零(disarmed-equiv)| `INS_Status >= READY` **+ holdoff §4.4** **+** D3 stateflow 接受 user 显式 re-arm |
| **`attitude_valid → 0`**(姿态失效;持续 ≥ N 帧 → FMS 路径;单帧 → Controller 路径)| F3-FMS-02:TR-03 → 若 INAIR M-07 LAND 后 M-01 LOCKDOWN;`cmd_mask=0` | F3-CTL-03 / F3-CTL-04:**5 ms 内**输出零推力 + integrator 清零(critical;比 FMS 路径快;两路径独立触发)| `attitude_valid == 1` **+ holdoff §4.4** + D3 接受 user re-arm(per D1 §4.4.4 TR-03 不可恢复) |
| **`position_valid → 0`**(位置失效;attitude / velocity 仍 1)| F3-FMS-03:若 mode ∈ {M-05/M-08/M-09/M-10} → M-04 ALTHOLD;否则 → M-03 STABILIZE;Command Shaper 不 emit BIT_POS;允许 BIT_VEL / BIT_ATT 等 | F3-CTL-01:L-01 旁通;integrator freeze + 清零;`vel_sp_inner = 0`(zero-frozen);L-02 改 use_external | `position_valid == 1` **+ holdoff §4.4**;mode 不自动恢复(user / mode 表显式重新进入,per D1 §4.4.4 TR-04 可恢复) |
| **`velocity_valid → 0`**(速度失效;attitude 仍 1)| F3-FMS-04:降级 M-03 STABILIZE;不 emit BIT_POS / BIT_VEL / BIT_ACC | F3-CTL-02:L-02 旁通;integrator 清零;cascade 降级 | `velocity_valid == 1` **+ holdoff §4.4** + user 重选 mode |
| **`heading_valid → 0`**(偏航失效;其他 OK)| F3-FMS-05:不 emit BIT_YAW;允许 BIT_YAW_RATE / BIT_ATT(roll/pitch only)| F3-CTL-05:yaw 角参考 hold-last 或 mag-only 退化;roll/pitch 正常 | `heading_valid == 1` **+ holdoff §4.4** |
| **`gps_valid → 0`**(GPS 失效;position_valid / velocity_valid 仍 1)| F3-FMS-06:Source Selector 屏蔽 mission 启动;**不**强制降级 mode(per D6 §4.5)| (无直接 — gate 在 FMS 层)| `gps_valid == 1`(若 mission 启动需求,Source Selector 重新 enable) |
| **`mag_valid → 0` / `baro_valid → 0`**(辅助源失效)| Safety Monitor 记录 degraded warning;**不**直接触发降级(`heading_valid` / `position_valid` 由 INS 内部仲裁;若关键位失效会另行触发上面行)| (无直接消费)| `*_valid == 1`(被动恢复;不需要 user 介入) |
| **`timestamp` stale > T_stale**(50 ms 默认)| F3-FMS-10:等价 `INS_Status < READY` → 走完全失效行 | (Controller 不消费 timestamp;但 cmd_mask=0 经 FMS 间接生效)| timestamp fresh + holdoff §4.4 |
| **多位同时失效**(e.g., attitude_valid + position_valid)| 取**最高级触发**(F3-FMS-02 attitude critical 优先);`failsafe_state` / `error_code` 反映该最高条;cmd_mask=0 | Controller 走最严厉路径(零推力 — F3-CTL-03/04 优先)| 所有失效位 == 1 + holdoff §4.4 |

**fallback hierarchy(致命度从高到低)**:

1. `attitude_valid == 0` 或 `INS_Status < READY` → 完全失效(零推力;不可恢复至需 re-arm)
2. `velocity_valid == 0`(attitude 仍 1)→ 降级到 M-03 STABILIZE
3. `position_valid == 0`(attitude / velocity 仍 1)→ 降级到 M-04 ALTHOLD
4. `heading_valid == 0`(其他 OK)→ yaw 子通道 hold;roll/pitch + thrust 正常
5. `gps_valid == 0`(其他 OK)→ mission 不可启动;不降级 mode
6. `mag_valid == 0` / `baro_valid == 0`(其他 OK)→ degraded warning;无 mode 影响

此层级与 [D1 §4.4.2 触发优先级](../D-fms/D1-fms-functional.md) 中 TR-03 / TR-04 的位置一致(TR-03 critical,TR-04 部分降级)。

### 4.4 Recovery / re-engagement holdoff

当某 validity 位由 0 → 1(恢复),消费侧**不得**立即 re-engage,以避免边界附近的 bouncing(在阈值附近反复触发降级 + 恢复造成模式抖动)。

| 维度 | 取值 / 政策 | 来源 / 备注 |
|---|---|---|
| **默认 holdoff 时长** | **100 ms**(= 5 个 Controller 5 ms 周期 = 1 个 INS 10 ms 周期)| F3 决策(Wave 9);可由 FMS_PARAM 配置(B3 增加 `ins_recovery_holdoff_s` 字段,可选)|
| **谁实施 holdoff** | FMS:Mode Manager(D3)在转换守卫上加 holdoff 计时器;Controller:Mask Resolver([E2 §4.1.1](../E-controller/E2-controller-structural.md))在每帧入口检测 + holdoff 计时 | 双侧实施;FMS holdoff 控制 mode 恢复,Controller holdoff 控制 loop re-engage |
| **holdoff 期间状态** | integrator **保持清零**(do NOT 恢复旧状态);loop 输出 = zero-frozen | per [E1 §4.4.4](../E-controller/E1-controller-functional.md) reset 策略;Controller 在 holdoff 结束后采用 [E1 §4.2.4](../E-controller/E1-controller-functional.md) bumpless transfer(integrator 重新初始化使新外环输出等于切换前内环测量值);具体公式由 [E3 §4.x](../E-controller/E3-loops-algorithm.md) 锁 |
| **Holdoff 期间 user 干预** | 允许 user 显式 re-arm 或 re-mode 请求,但 D3 守卫仍要求 holdoff 完成后才接受(防止过早恢复)| D1 §4.4.4 + A6 §4.5.3 |
| **不可恢复触发**(per D1 §4.4.4)| TR-03(INS attitude critical 失效)即使 attitude_valid 恢复 + holdoff 满足,**仍要求 user 显式 disarm 后重新 arm**;TR-04(GPS 失效部分降级)允许 holdoff 后由 user 切回原 mode | D1 §4.4.4 |
| **Holdoff 与 D1 §4.4.3 debounce 的关系** | debounce(进入 fallback 的判据)≠ holdoff(退出 fallback 的判据)。debounce ≈ N 帧持续 0 才触发(防瞬时);holdoff ≈ 100 ms 持续 1 才允许恢复(防 bounce)。两者是**单向触发 + 回滞恢复**的对称组件 | D1 §4.4.3 锁 debounce;F3 锁 holdoff;两者由 FMS_PARAM `ins_unready_timeout_s` + `ins_recovery_holdoff_s` 各自配置 |

**bumpless transfer 协议**(holdoff 结束后):

1. integrator 状态保持 0(不复用 0 之前的旧 integrator state;per [E1 §4.4.4](../E-controller/E1-controller-functional.md));
2. setpoint 沿 D4 整形(per D6 §4.6.2 setpoint 字段值契约 — 经 rate-limited shape);
3. cascade 重连接(per [E1 §4.2.4](../E-controller/E1-controller-functional.md) 不变量 3:新外环 integrator 重初始化使其内环 setpoint 等于切换前的内环测量值;具体由 E3 锁公式)。

### 4.5 Timestamp staleness 检测

#### 4.5.1 检测谁

仅 **FMS Source Selector** 消费 `INS_Out_Bus.timestamp` 做 staleness 判定。Controller **不**消费 timestamp(per [A7 §4.5](../A-architecture/A7-time-conventions.md):Controller 用固定步长 5 ms 派生 dt;[A3 §4.5.3 注](../A-architecture/A3-module-boundaries.md))。

理由:
- INS 是 firmware 10 ms 周期生产;FMS 是 20 ms 周期消费,中间经 RB-04 sample-time-based downsample,任何调度漂移都会反映在 timestamp 上 → FMS 必须主动检测。
- Controller 是 5 ms 周期消费,中间经 RB-03 ZOH;若 INS 卡死,ZOH 会持续输出旧值 → Controller 自身无法分辨;此分辨任务委托给 FMS staleness gate + cmd_mask 联动(FMS 见 stale → emit cmd_mask=0 → Controller 走 D6 MX-1 disarmed-equiv)。

#### 4.5.2 staleness threshold

| 项 | 默认值 | 备注 |
|---|---|---|
| `T_stale`(stale 判据)| **50 ms**(= 5 × INS 10 ms 周期)| 可由 FMS_PARAM 配置(B3 增加 `ins_stale_threshold_s`,可选);允许 4 个 INS 周期的调度抖动余量;第 5 个周期内必须出现新 timestamp,否则视为 stale |
| **stale 检测算子** | `now_ms - INS_Out_Bus.timestamp > T_stale` | `now_ms` 来自 FMS 自身 step interface timestamp(per A7 §4.4 单调);若 `INS_Out_Bus.timestamp` wrap,按 [A7 §4.3](../A-architecture/A7-time-conventions.md) wrap-aware 减法处理(经 [A8 §4.2.5](../A-architecture/A8-shared-library-roster.md) `Stale_Detector` 共享原语)|
| **stale 后行为** | F3-FMS-10:等价 `INS_Status < READY` → 走完全失效行(F3-FMS-01)| Source Selector 设置内部 `ins_stale_latch=1` → Mode Manager 拒 arm + Command Shaper emit cmd_mask=0;Controller 间接通过 cmd_mask=0 进入 disarmed-equiv |
| **stale 恢复** | timestamp 恢复 fresh 持续 ≥ holdoff §4.4 后,清 latch | 防 stale ↔ fresh 抖动 |

#### 4.5.3 timestamp 单调性 / wrap

由 [A7 §4.4](../A-architecture/A7-time-conventions.md) 保证:`INS_Out_Bus.timestamp` 单调非降(允许重复值,不允许回退);wrap 在 uint32 ms ≈ 49.7 天;模型仓侧不期望连续运行至 wrap,F3 不展开 wrap-edge 行为(由 [A7 §4.3](../A-architecture/A7-time-conventions.md) 处理)。

### 4.6 Per-subsystem 规则速查

按消费方分组的快查表(供下游 D3 / D4 / E1 / E3 / G3 / H1 / H4 实现 + 测试时对照):

#### 4.6.1 FMS Source Selector

- 消费字段:`timestamp`(staleness)+ `INS_FLAG_BIT_GPS_VALID`(mission availability)
- 关键规则:F3-FMS-06、F3-FMS-10
- holdoff 实施:在 stale latch / GPS-lost latch 上加 holdoff §4.4

#### 4.6.2 FMS Mode Manager(D3)

- 消费字段:`INS_Status`、`INS_FLAG_BIT_ATTITUDE_VALID`、`INS_FLAG_BIT_POSITION_VALID`、`INS_FLAG_BIT_VELOCITY_VALID`
- 关键规则:F3-FMS-01..F3-FMS-09(arm guard + mode entry guard + 失效降级)
- holdoff 实施:在 mode transition 守卫上加 holdoff;不可恢复触发(F3-FMS-01 / F3-FMS-02)要求 user 显式 re-arm

#### 4.6.3 FMS Mission/Auto Manager

- 消费字段:`position_lla`、`position_ned_m`、`INS_FLAG_BIT_POSITION_VALID`、`INS_FLAG_BIT_GPS_VALID`
- 关键规则:autoarm 条件 = position_valid && gps_valid;waypoint 算术 gate position_valid;geofence 检测 = TR-07(per D1 §4.4.1)
- holdoff 实施:mission 启动条件加 holdoff(GPS lock 后等 100 ms 才允许 takeoff)

#### 4.6.4 FMS Safety Monitor

- 消费字段:**所有 validity 位 + INS_Status + timestamp**(全集)
- 关键规则:TR-03 / TR-04 / TR-08(arm precondition 拒绝)的触发判据
- holdoff 实施:由 [A8 §4.2.5](../A-architecture/A8-shared-library-roster.md) `Stale_Detector` + Validity_AND 原语承载

#### 4.6.5 FMS Command Shaper(D4)

- 消费字段:`velocity_ned_mps`(闭环参考整形)、`quat_ned_to_b`(frame transform)、`INS_FLAG_BIT_VELOCITY_VALID`、`INS_FLAG_BIT_ATTITUDE_VALID`
- 关键规则:emit cmd_mask 时遵守 D6 §4.5 良民规则(echo 在 F3-FMS-01..F3-FMS-05);失效情境下不 emit 对应 BIT_*

#### 4.6.6 FMS Output Assembler

- 消费字段:`INS_Status`、`INS_FLAG_BIT_*` 位(passthrough echo,若 [A3 §4.4.6.5](../A-architecture/A3-module-boundaries.md) echo 字段在 B1 锁定)
- 关键规则:**纯 passthrough**(per A3 §4.6 passthrough-direct);Output Assembler 不修改 validity,只把 INS validity 位 echo 到 `FMS_Out_Bus.ins_*` 可选字段(若存在)
- holdoff 实施:无(passthrough 透明)

#### 4.6.7 Controller L-01 Position

- 消费字段:`position_ned_m[3]`、`INS_FLAG_BIT_POSITION_VALID`
- 关键规则:F3-CTL-01;5 ms 即时 gate;失效→旁通 + integrator 清零

#### 4.6.8 Controller L-02 Velocity

- 消费字段:`velocity_ned_mps[3]`、`INS_FLAG_BIT_VELOCITY_VALID`、`acc_b_mps2[3]`(可选 FF)
- 关键规则:F3-CTL-02;5 ms 即时 gate

#### 4.6.9 Controller L-03 Attitude(critical)

- 消费字段:`quat_ned_to_b[4]`、`INS_FLAG_BIT_ATTITUDE_VALID`、`INS_FLAG_BIT_HEADING_VALID`(yaw 子轴)
- 关键规则:F3-CTL-03(attitude critical 零推力);F3-CTL-05(yaw 子轴 hold)

#### 4.6.10 Controller L-04 Rate(critical)

- 消费字段:`ang_rate_b_radps[3]`、`INS_FLAG_BIT_ATTITUDE_VALID`(rate gate 复用 attitude_valid,Phase 2)
- 关键规则:F3-CTL-04(critical 零推力,与 L-03 联动)

#### 4.6.11 ins_stub passthrough(F2 §4.3)

- 由 [F2 §4.3](F2-ins-stub-structural.md) 字段映射表落地;F3 仅声明:ins_stub 是 producer 侧,产出**符合本 F3 fallback 表的全部 validity 模式**(经 F1 §4.3.5 transient timeline + F2 §4.4 init transient chart)
- harness scenario(H1 / H4)通过 ins_stub 的 health knob / transient timing 触发 F3 fallback 路径,作为 oracle

#### 4.6.12 harness / G3 logsout

- 消费:**所有字段必记**(per [G3](../G-harness/G3-logging.md) 最小信号集要求,架构 v1 §13);特别记录 INS validity 位 + cmd_mask + mode/state 联动,以便事后 trace 任何 fallback 触发
- F3 不规定 logsout schema(由 G3 锁);只声明:F3 §4.3 表的每一行触发都必须在 logsout 中可观测

### 4.7 边界场景

#### 4.7.1 冷启动(boot)

- 时刻 t=0:firmware INS 输出 `INS_Status=NOT_READY` + `INS_Flag=全 0`(per A6 §4.5.3 reset-time 安全态);ins_stub 在 NOISY 模式下走 F1 §4.3.5 / F2 §4.4 transient timeline(NOT_READY → INITIALIZING → READY,各位逐步翻转)
- FMS 行为:停留 M-01 DISARMED / IDLE(per D1 §4.4 INIT super-state + A6 §4.5.3 第 2 项);拒 arm(F3-FMS-07);不进 anywhere(F3-FMS-08 / F3-FMS-09 守卫)
- Controller 行为:cmd_mask=0(FMS 在 INIT 阶段 emit) → 走 MX-1 disarmed-equiv(零推力,A6 §4.4.2);per-loop gate 全部触发(F3-CTL-01..F3-CTL-04)— 但因 cmd_mask=0,本就走 disarmed,gate 触发是冗余防御
- 冷启动结束:`INS_Status==READY` + 所有关键位 == 1 + holdoff(§4.4)→ Mode Manager 接受 user arm 请求

#### 4.7.2 瞬时 glitch(1-2 帧 validity 位 0 然后恢复)

- FMS 路径(N=`ins_unready_timeout_s` × 50 帧)**不触发**(debounce 吞掉 < N 的 glitch);`failsafe_state` / `error_code` 不变
- Controller 路径(无 debounce)在该 1-2 帧内**临时旁通**对应环路 + integrator 清零;glitch 结束后 holdoff 100 ms 才允许 re-engage(§4.4)→ 等价"瞬时小坑;cascade 短暂 zero-frozen;无可观测行为变化(假设 holdoff 内无 user 输入)"
- 边界后果:Controller 在 100 ms holdoff 内输出可能比 glitch 前 slightly 不同(integrator 清零会丢失轻微 windup),但 bumpless transfer 协议保证不出现可观测阶跃

#### 4.7.3 持续失效(>N 帧)

- FMS 路径触发降级 / failsafe(per F3-FMS-01..F3-FMS-05);`failsafe_state` 设置;若不可恢复 → 锁在 LOCKDOWN
- Controller 路径长期处在旁通 / disarmed-equiv;cmd_mask 由 FMS 控制,Controller 透明
- 恢复后:holdoff §4.4 完成才允许 mode 切回;不可恢复触发(F3-FMS-01 / F3-FMS-02)需 user 显式 disarm + 重新 arm(per D1 §4.4.4)

#### 4.7.4 多位同时失效(simultaneous drops)

- **裁决**:取**致命度最高**的失效位的 fallback 路径;`failsafe_state` / `error_code` 反映该最高条;后续位仅在 logsout 记录,不二次降级
- 致命度顺序(per §4.3 fallback hierarchy):
  1. `attitude_valid==0` 或 `INS_Status<READY`(critical)
  2. `velocity_valid==0`
  3. `position_valid==0`
  4. `heading_valid==0`
  5. `gps_valid==0`
  6. `mag_valid==0` / `baro_valid==0`
- 例:`attitude_valid==0 && position_valid==0` 同帧 → FMS 走 F3-FMS-02(critical;FS-04 LOCKDOWN);Controller 走 F3-CTL-03(零推力);position_valid==0 仅在 logsout 记录,不触发额外 M-04 降级

#### 4.7.5 INS_Status 与 INS_Flag 不一致(firmware 内部矛盾)

- 情境:firmware 上报 `INS_Status==READY` 但 `INS_FLAG_BIT_ATTITUDE_VALID==0`(逻辑上矛盾;按 B2 §4.4.10 注:`INS_Status>=READY` 与各 flag 位独立检查)
- 裁决:**flag 位优先**(更细粒度;F3-FMS-02 / F3-CTL-03 触发);INS_Status==READY **不**抑制 flag 位的失效行为
- 反向(`INS_Status==NOT_READY` 但所有 flag 位==1):**INS_Status 优先**(更严格;F3-FMS-01 触发)
- 总规则:**不一致时取更严厉的那一边**

#### 4.7.6 firmware INS 字段语义漂移(B5 镜像变更)

- 情境:[B5](../B-contracts/B5-ins-bus-mirror.md) mirror artifact 因 firmware tip 变更而更新(新增字段 / 删除字段 / 字段语义变 / 位号变)
- 行为:F3 §4.1 / §4.3 表必须在 [B5](../B-contracts/B5-ins-bus-mirror.md) reviewed 同 batch 重新校验;若 fallback 表行需要更新,F3 通过 §8 变更日志记录 + INDEX 决策日志登记(per [B5 §4.8](../B-contracts/B5-ins-bus-mirror.md) F-area 下游耦合)
- 防漂移工具:[B4](../B-contracts/B4-contract-diff.md) + [I3](../I-tooling/I3-contract-diff.md) 契约 diff 在每次 firmware export 触发;若发现 INS_Out_Bus 字段集与本 F3 §4.1 表不一致,export pipeline FAIL

### 4.8 Cross-references

| 关联文档 | F3 关联章节 | 关系类型 | 备注 |
|---|---|---|---|
| [A3 §4.8](../A-architecture/A3-module-boundaries.md) | §4.1 | **权威源**(消费侧依赖矩阵集中表)| F3 §4.1 = A3 §4.8 的可执行扩展;若 A3 §4.8 矩阵更新,F3 §4.1 必须同步 |
| [D1 §4.4](../D-fms/D1-fms-functional.md) | §4.2.1 / §4.3 / §4.7 | **echo + 扩展**(TR-03 / TR-04 详细,debounce / hysteresis;F3 不修改触发逻辑,只把 INS 相关条款重排为消费规则视角)| |
| [D6 §4.5](../D-fms/D6-fms-controller-interface.md) | §4.2.1 / §4.3 | **echo**(INS-invalid 情境下 FMS cmd_mask emission policy)| F3 把 D6 §4.5 与 D1 §4.4 拼合 |
| [E1 §4.5](../E-controller/E1-controller-functional.md) | §4.2.2 / §4.3 / §4.6.7..§4.6.10 | **echo**(Controller per-loop gating;zero-frozen / hold-last 默认策略)| F3 不修改 loop 算法,只锁定 validity-drop 期间的功能意图 |
| [F2 §4.4](F2-ins-stub-structural.md) | §4.7.1 | **producer-side**(ins_stub init transient chart 的 supply 行为是 F3 cold-start 行为的对偶)| F3 是 consumer 侧;F2 是 producer 侧;两者对 transient timeline 的描述应一致 |
| [B5 §4.8](../B-contracts/B5-ins-bus-mirror.md) | §4.7.6 | **下游纪律**(B5 mirror 变更 → F3 fallback 表必须重新校验)| 与 B5 §4.8 F-area 下游耦合纪律对齐 |
| [H4](../H-verification/H4-fault-catalog.md)(Wave 11) | §4.6.11 | **下游消费**(H4 故障目录的传感器 / 估计器类故障会触发 F3 §4.3 表中各行 fallback)| F3 §4.3 表是 H4 场景的 oracle;H4 必须覆盖每个 §4.3 行 |
| [H1](../H-verification/H1-scenario-catalog.md)(Wave 11) | §4.7 | **下游消费**(H1 验证场景目录的边界场景章节会引用 F3 §4.7)| H1 必须覆盖 F3 §4.7.1..§4.7.5 五个边界场景 |
| [G3](../G-harness/G3-logging.md)(Wave 10) | §4.6.12 | **下游消费**(G3 logsout 必须记录 F3 §4.1 全字段 + cmd_mask + mode/state)| G3 logsout 是 F3 §4.3 表行为的可观测性载体 |

## 5. 已知风险与悬而未决问题

- **R-F3-1 — `T_stale=50 ms` 数值未由 PARAM 落定**
  - 影响:F3-FMS-10(timestamp staleness)依赖此阈值;若实施时发现 firmware INS 调度抖动 > 50 ms,FMS 会误触发完全失效;若 < 50 ms 阈值过松,真正卡死的 INS 会延迟检测
  - 处置:F3 默认 50 ms;[B3](../B-contracts/B3-parameter-schema.md) 在 Wave 9/10 决议是否新增 `ins_stale_threshold_s` 字段(可选,runtime-tunable);H1 / H4 需提供 sensitivity 场景验证

- **R-F3-2 — `ins_recovery_holdoff_s=100 ms` 数值未由 PARAM 落定**
  - 影响:同 R-F3-1;若 holdoff 过短 → bouncing;过长 → 恢复迟钝
  - 处置:F3 默认 100 ms = 5 Controller cycles = 1 INS cycle;[B3](../B-contracts/B3-parameter-schema.md) 在 Wave 9/10 决议是否新增 `ins_recovery_holdoff_s` 字段(可选)

- **R-F3-3 — heading_valid 失效时的 mag-only fallback 算法未定**
  - 影响:F3-CTL-05 列出"yaw 角参考 hold-last 或 mag-only 退化",但具体算法由 [E3](../E-controller/E3-loops-algorithm.md) 锁;F3 仅锁功能意图
  - 处置:E3(Wave 9/10)闭合;F3 与 E3 在同 Wave 互引

- **R-F3-4 — INS_Status 与 INS_Flag 不一致裁决依赖 firmware 行为假设**
  - 影响:§4.7.5 假设"firmware 不应同时 emit READY + flag.attitude_valid==0",但 firmware tip 行为未在本环境核对(`<pending hash>`)
  - 处置:[B5](../B-contracts/B5-ins-bus-mirror.md) 镜像 PR review 时核对;若 firmware 确实可能 emit 不一致组合,F3 §4.7.5 的"flag 优先 / status 优先"规则按现状已涵盖,无需改

- **R-F3-5 — Multiple-drop 同帧裁决的 logsout 可观测性依赖 G3**
  - 影响:§4.7.4 要求"次低位仅 logsout,不二次降级";若 G3 未把所有 INS_Flag 位放进 logsout 最小信号集,debug 时找不到次低位失效证据
  - 处置:[G3](../G-harness/G3-logging.md)(Wave 10)必须把 INS_Flag 全位 + INS_Status + cmd_mask + mode + failsafe_state 全部放入最小信号集(per §4.6.12)

- **R-F3-6 — D3 / D4 hand-off 的 holdoff 计时器未具体落地**
  - 影响:§4.4 要求"FMS 由 D3 实施 holdoff",但 D3 状态机草图未见 holdoff 计时器;D4 整形器同理
  - 处置:D3 / D4(Wave 8 已 reviewed)若已含此计时器,F3 引用即可;若未含,D3 / D4 在 Wave 9/10 增补;F3 在变更日志记录

- **R-F3-7 — firmware INS commit hash `<pending hash>` 占位**
  - 影响:F3 引用 `INS_Out_Bus` / `INS_Status` / `INS_Flag` 的 firmware-canonical 字段名 / 位号都依赖 [B5](../B-contracts/B5-ins-bus-mirror.md) mirror,而 B5 §3 也用 `<pending hash>` 占位
  - 处置:与 F-area / B5 同 batch 占位约定;FMT-Firmware 仓挂载后由 B5 镜像 PR + INDEX 决策日志补齐;F3 在变更日志记录

## 6. 退出条件复核

对照 [`00-design-plan.md`](../00-design-plan.md) §4.F F3 行的退出条件:**"FMS/Controller 对 `INS_Out_Bus` 字段的依赖矩阵、validity 处理规则"**。逐条复核:

| # | 退出条件原文 | 本文档依据 | 状态 |
|---|---|---|---|
| 1 | FMS 对 `INS_Out_Bus` 字段的依赖矩阵 | §4.1 per-field 消费方矩阵 FMS 列(echo + 扩展 [A3 §4.8](../A-architecture/A3-module-boundaries.md));§4.6.1..§4.6.6 FMS 6 子模块逐一规则速查 | 满足 |
| 2 | Controller 对 `INS_Out_Bus` 字段的依赖矩阵 | §4.1 per-field 消费方矩阵 Controller 列;§4.6.7..§4.6.10 L-01..L-04 逐环规则速查 | 满足 |
| 3 | validity 处理规则(FMS) | §4.2.1 FMS 规则表 F3-FMS-01..F3-FMS-10(覆盖 INS_Status / 5 个关键 INS_Flag 位 / arm guard / mode entry guard / staleness)| 满足 |
| 4 | validity 处理规则(Controller) | §4.2.2 Controller 规则表 F3-CTL-01..F3-CTL-08(覆盖 4 个环路 + MX-3 拒绝 + INS_Status 间接消费 + timestamp 不消费)| 满足 |
| 5(隐含)| Fallback 政策权威表 | §4.3 fallback 政策表(8 类失效情境 + 多位同时失效 + fallback hierarchy)| 满足 |
| 6(隐含)| Recovery / re-engagement 政策 | §4.4 holdoff(默认 100 ms)+ bumpless transfer 协议 | 满足 |
| 7(隐含)| Staleness 检测 | §4.5 timestamp staleness(默认 50 ms,FMS Source Selector 实施)| 满足 |
| 8(隐含)| 边界场景闭合 | §4.7 5 个边界场景(冷启动 / 瞬时 glitch / 持续失效 / 多位同时 / status-flag 不一致)+ §4.7.6 firmware 漂移 | 满足 |

## 7. 下游影响

按 [`01-design-relationships.md`](../01-design-relationships.md) F3 的出边(F3 是 F 区终点,无 F 内出边;但 F3 是其他 Wave 11 / 实现阶段下游的 oracle):

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [H1 验证场景目录](../H-verification/H1-scenario-catalog.md)(Wave 11)| ⇢ | §4.7 5 个边界场景 → H1 必须覆盖;§4.3 fallback 表每行 → H1 / H4 oracle |
| [H4 故障注入目录](../H-verification/H4-fault-catalog.md)(Wave 11)| ⇢ | §4.6.11 ins_stub passthrough 列 + §4.3 fallback 表 → H4 注入场景必须覆盖每行触发条件;`H4 ⇢ F1` 关系扩展为"H4 ⇢ F3 验证 oracle" |
| [H2 验证指标](../H-verification/H2-metrics.md)(Wave 11)| ⇢ | §4.4 holdoff 100 ms / §4.5 staleness 50 ms 的数值阈值是 H2 量化指标的 anchor(per H2 退出条件"每个 H1 场景给出具体数值阈值")|
| [G3 日志](../G-harness/G3-logging.md)(Wave 10)| ⇢ | §4.6.12 + §5 R-F3-5:G3 最小信号集必须含 INS_Flag 全位 + INS_Status + cmd_mask + mode + failsafe_state |
| [B3 Parameter schema](../B-contracts/B3-parameter-schema.md)(Wave 已 reviewed,但 R-F3-1 / R-F3-2 可能要求增补)| ⇢ | §5 R-F3-1 / R-F3-2:可选 `ins_stale_threshold_s` / `ins_recovery_holdoff_s` 字段(若 B3 决议增补)|
| [E3 Controller 算法](../E-controller/E3-loops-algorithm.md)(Wave 9)| ⇢ | §4.4 bumpless transfer 协议 + §5 R-F3-3 mag-only fallback;E3 落具体公式 |
| [I5 sim runner](../I-tooling/I5-sim-runner.md)(Wave 10)| ⇢ | §4.7.1..§4.7.5 边界场景 → I5 批量回归脚本的 fixture 列表 |
| [实现阶段] FMS Simulink / Stateflow | (实现)| §4.2.1 / §4.6.2 → D3 stateflow 守卫 + holdoff 计时器实现规范 |
| [实现阶段] Controller Simulink | (实现)| §4.2.2 / §4.6.7..§4.6.10 → Mask Resolver + 各环 valid_gate 端口实现规范 |

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-09 | author | 初稿;基于 [A3 §4.8](../A-architecture/A3-module-boundaries.md) 矩阵、[D1 §4.4](../D-fms/D1-fms-functional.md) TR-03/TR-04、[D6 §4.5](../D-fms/D6-fms-controller-interface.md) emission policy、[E1 §4.5](../E-controller/E1-controller-functional.md) per-loop gating、[F1 §4.3.5](F1-ins-stub-functional.md) / [F2 §4.4](F2-ins-stub-structural.md) transient timeline 拼合;锁定 holdoff=100 ms / staleness=50 ms 默认值 |

## Self-check

- [x] frontmatter 完整,字段值合法
- [x] 上游文档全部存在且 status ≥ reviewed,或为同 co-seal batch 兄弟(已在 §3 注明) — A1 / A3 / A6 / A7 / B1 / B2 / B5 / D1 / D6 / E1 / F1 / F2 全部已 reviewed;D3 / D4 在 Wave 8 同 batch reviewed(F3 是其下游 Wave 9,不存在 forward-cite 风险);架构 v1 是基线
- [x] 退出条件逐条复核完成,每条均给出依据 — §6 表覆盖 8 条(2 条退出条件原文 + 6 条隐含)
- [x] 引用路径全部可点击访问 — 全部相对路径,均指向 `docs/design/<area>/` 或 `docs/architecture/`
- [x] 不存在 RULES §5 禁则中的内容 — 无 .slx 截图占位、无可执行 .m 代码、不重复 firmware 实现细节、不重复 bus/enum 字段表(只引用 B1 / B2 / B5)
- [N/A] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记 — contract_impact=**no**,N/A
- [N/A] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径 — F3 不直接镜像 firmware;通过 [B5](../B-contracts/B5-ins-bus-mirror.md) mirror artifact 间接消费;hash `<pending hash>` 占位与 F-area / B5 同 batch 约定
- [x] 下游影响已沿关系图识别完毕 — §7 列 H1 / H4 / H2 / G3 / B3 / E3 / I5 + 实现阶段 FMS / Controller 共 9 项
- [x] 文档不超出本工作项范围(无越权设计)— 不重定义 INS_Out_Bus schema / INS_Status / INS_Flag / B5 mirror 流程 / D1 失效触发 / D6 cmd_mask 互斥 / Controller 数学公式;仅锁消费侧 fallback 规则
