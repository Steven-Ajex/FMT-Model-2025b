---
work_item: G1
title: MIL 顶层结构设计
upstream: ["架构 v1", "A1", "A2", "A3", "A4", "A5", "A6", "A7", "A8", "B1", "B2", "B3", "C1", "C2", "C4", "D1", "D2", "E1", "E2", "F1", "F2"]
contract_impact: no
status: draft
authored_at: 2026-05-10
last_reviewed_at:
reviewer_verdict: none
---

# G1 MIL 顶层结构设计

## 1. 目的

为 FMT-Model-2025b Phase 2 多旋翼 MIL 闭环锁定**顶层 harness 模型 `model/harness/mil/mil_top.slx`** 的子系统拓扑、模型引用策略、以及变体接入点(variant attachment points):把 `Plant.slx` / `FMS.slx` / `Controller.slx` 三份独立 model-reference 模块、`ins_stub` 子系统、若干外部命令源(Pilot / GCS / Auto / Mission)与一份 Environment 配置在 harness 顶层装配为一个端到端可仿真的闭环。本文件**只**锁定顶层 harness 结构;G2(Solver / Sample Time pane)、G3(logsout 信号集)、G4(Pilot_Cmd 时间序列)、E5(Controller 5 ms 子预算)由各自工作项 forward-cite 闭合。

## 2. 范围

**在范围:**

- §4.1 顶层子系统拓扑(Mermaid):`Pilot_Cmd_Source` / `GCS_Cmd_Source` / `Auto_Cmd_Source` / `Mission_Data_Source` / `Environment_Info_Source` / `States_Init_Source` / `Plant`(model reference)/ `FMS`(model reference)/ `Controller`(model reference)/ `ins_stub`(plain Subsystem)— 共 9 个顶层块 + logsout outport 集合
- §4.2 三份顶层模型的 **Model Reference** 实例化方式与参数传递(per [A5 §4.1.2](../A-architecture/A5-variant-strategy.md))
- §4.3 顶层 5 个 variant 接入点(VP-Vehicle / VP-INS-Variant / VP-Plant-States-Bypass / VP-Control-Out-Echo / VP-Harness-Variant)与各自的控制变量、Phase 2 默认值、forward path
- §4.4 init / reset wiring(`model_init` 执行链 + harness reset 入口)— 仅 declare,数值与具体子状态 reset 由各模块自身设计
- §4.5 Solver 与 rate-transition placement **声明**(forward-cite [G2](G2-rate-scheduling.md))
- §4.6 logsout port 装配规则**声明**(forward-cite [G3](G3-logging.md))
- §4.7 Pilot_Cmd_Source 顶层接入点**声明**(forward-cite [G4](G4-pilot-injection.md))
- §4.8 跨工作项交叉引用与 Wave 10 sibling 协调

**不在范围(由其他工作项处理):**

- 各模块顶层 `.slx` 内部子系统拓扑 — Plant 由 [C2](../C-plant/C2-plant-structural.md);FMS 由 [D2](../D-fms/D2-fms-structural.md);Controller 由 [E2](../E-controller/E2-controller-structural.md);ins_stub 由 [F2](../F-ins-contract/F2-ins-stub-structural.md)
- bus 字段 / enum 数值 / parameter schema — B 区([B1](../B-contracts/B1-bus-inventory.md) / [B2](../B-contracts/B2-enum-inventory.md) / [B3](../B-contracts/B3-parameter-schema.md))已锁,本文件只引用
- Solver 配置项(fixed-step base rate / 离散积分器 / Sample Time 着色规则)— 由 [G2](G2-rate-scheduling.md)(Wave 10 sibling)
- logsout 信号集 / 命名 / 单位 metadata / 可视化目录 — 由 [G3](G3-logging.md)(Wave 10 sibling)
- Pilot_Cmd 时间序列格式 / 场景 DSL / scenario 注入路径 — 由 [G4](G4-pilot-injection.md)(Wave 10 sibling)
- Controller 5 ms 内子预算 / 各环 µs 分配 / 热点禁用项 — 由 [E5](../E-controller/E5-performance-budget.md)(Wave 10 sibling)
- 各 PARAM 字段具体数值与多旋翼 leaf 默认值 — 由 [B3](../B-contracts/B3-parameter-schema.md) / [C3](../C-plant/C3-multicopter-leaf.md) / [D5](../D-fms/D5-multicopter-leaf.md) / [E4](../E-controller/E4-multicopter-leaf.md)
- ins_stub 内部块 / 字段映射 / RNG sub-seed / knob carrier — 由 [F1](../F-ins-contract/F1-ins-stub-functional.md) / [F2](../F-ins-contract/F2-ins-stub-structural.md)
- harness 层 SIH / HIL 拓扑(Phase 3+)— 仅在 §4.3 VP-Harness-Variant 留 forward path,Phase 2 不实施
- init 脚本(`FMT_Model_Init.m`)的 IO / 加载顺序 / worked example — 由 [I1](../I-tooling/I1-init-script.md);本文件只声明 harness `model_init` 调用义务
- 仿真运行 / 批量回归脚本 CLI — 由 [I5](../I-tooling/I5-sim-runner.md)
- 验证场景目录 / 指标 / 回归基线 — 由 [H1](../H-verification/H1-scenario-catalog.md) / [H2](../H-verification/H2-metrics.md) / [H3](../H-verification/H3-regression-baseline.md)
- 故障注入端口 schema — 由 [H4](../H-verification/H4-fault-catalog.md)(Wave 11),G1 仅在 §4.3 VP-Plant-States-Bypass 旁标注 H4 fault input port 的存在

## 3. 依赖

| 上游 | 引用位置 | 用途 |
|---|---|---|
| 架构 v1 §6 仓库树 / §11 layering / §15.1 MIL / §16 smallest slice | [架构 v1](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | `model/harness/mil/`(顶层落地路径)+ `model/harness/ins_stub/`(ins_stub 落地路径)+ Phase 2 smallest slice 6 项 minimum scope(arm → takeoff/hold → manual or position hold → land/disarm)+ MIL minimum scope demand(FMS + Controller + Plant + ins_stub) |
| [A1 v1 评审与缺口闭合](../A-architecture/A1-v1-review.md)(reviewed 2026-05-05) | A1 §4.2 §10.4 / §17 risk 4(MIL stub realism)/ §17 OQ4 | G1 是 §10.4 ins_stub harness 接入点的最终落地者;risk 4 mitigation 需要 G1 在顶层提供 IDEAL/NOISY 切换可达性 |
| [A2 命名与目录约定](../A-architecture/A2-naming-conventions.md)(reviewed 2026-05-05) | A2 模型 / 子系统 / variant 控制变量命名规则 | §4.1 顶层块名 / §4.3 variant 控制变量名(`MIL_VEHICLE` / `INS_VARIANT_MODE` / `VP_PLANT_STATES_BYPASS` / `VP_CONTROL_OUT_ECHO` / `HARNESS_VARIANT`)沿用 A2 约定 |
| [A3 模块边界信号清单](../A-architecture/A3-module-boundaries.md)(reviewed 2026-05-07) | A3 §4.3 / §4.4 / §4.5 / §4.6 三模块输入 / 输出 bus 字段级清单 + 字段方向 / Reset 类别 / 单位坐标系标注 | §4.1 三模块端口集与 wiring 方向 = A3 字段方向矩阵的顶层装配视角投影 |
| [A4 跨速率边界设计](../A-architecture/A4-rate-boundaries.md)(reviewed 2026-05-05) | A4 §4.2 RB-01..RB-07 / §4.5 整数倍同相对齐 / §4.7 边界文档化模板 | §4.5 G1 顶层在 RB 之间的 placement 声明(具体 Solver 由 G2);§4.1 wiring 引用 RB-01..RB-06 |
| [A5 变体策略设计](../A-architecture/A5-variant-strategy.md)(reviewed 2026-05-05) | A5 §4.1.2 三模块 = Model Reference / §4.5 Controller harness-only `Plant_States_Bus` variant 接入框架 / §4.6 codegen 影响 | §4.2 Model Reference 实例化与参数传递;§4.3 VP-Plant-States-Bypass 是 A5 §4.5 的物理拓扑落地 |
| [A6 Init/Reset 状态契约](../A-architecture/A6-init-reset-contract.md)(reviewed 2026-05-05) | A6 §4.2.1 init 无参签名 / §4.3.1 TS-MIL-HARNESS / §4.5.1 init 调用顺序 / §4.5.2 global vs local reset / §4.7.1 MIL-firmware 非对称项 | §4.4 init / reset wiring;§4.4.2 顺序 = A6 §4.5.1 推荐顺序;§4.4.3 TS-MIL-HARNESS 入口 = A6 §4.3.1 触发源代号 |
| [A7 时间与时间戳约定](../A-architecture/A7-time-conventions.md)(reviewed 2026-05-05) | A7 §4.1 uint32 ms / §4.2 zero-from-boot / §4.4 单调 / §4.8 timestamp 用途 | §4.4 init 时刻 t=0 与 timestamp 派生由 A7 锁;§4.5 整数倍同相起步 = A4 §4.5 + A7 §4.2 联合保证 |
| [A8 共享库块清单与归属](../A-architecture/A8-shared-library-roster.md)(reviewed 2026-05-05) | A8 共享库块台账(Bus_Selector / Validity_AND / Health_Combine 等) | §4.1 / §4.6 G1 装配中如需 bus split / merge 优先消费 A8 共享块;不重复定义 |
| [B1 Bus 清单与 schema](../B-contracts/B1-bus-inventory.md)(reviewed 2026-05-07) | B1 全部 16 firmware-mirrored bus + 3 内部 bus + sub-schema | §4.1 wiring 中所有 bus 名引用 B1;G1 不重述字段 |
| [B2 Enum 清单与数值锁定](../B-contracts/B2-enum-inventory.md)(reviewed 2026-05-07) | B2 14 contract-locked enums + cmd_mask bit 位号 | §4.1 wiring 标注 enum 类型字段时引用 B2 |
| [B3 Parameter schema](../B-contracts/B3-parameter-schema.md)(reviewed 2026-05-07) | B3 PLANT_PARAM / FMS_PARAM / CONTROL_PARAM 116 + 5(E3 sealed amendment)字段 | §4.4.4 `States_Init_Source` 从 PLANT_PARAM 派生(per A6 §4.4.2 Plant_States_Bus PARAM 类别行) |
| [C1 Plant 功能设计](../C-plant/C1-plant-functional.md)(reviewed 2026-05-08) | C1 §4.7 reset 类别 / §4.7.1 motor states ZERO / 7 类物理 | §4.4.4 `States_Init_Source` 字段集来源(C1 锁 init 字段集) |
| [C2 Plant 结构设计](../C-plant/C2-plant-structural.md)(reviewed 2026-05-08) | C2 §4.1 Plant 顶层 .slx 端口集(3 input bus + 1 reset trigger + 1 H4 input + 7 output bus + 0 init args)/ §4.6 Reset Port / §4.7 H4 input port | §4.1 Plant block 端口集精确匹配 C2 §4.1;§4.4.3 reset trigger 进入 Plant Reset Port |
| [C4 Plant 数值与积分](../C-plant/C4-numerics.md)(reviewed 2026-05-09) | C4 RK4 fixed-step 1 ms / 17 连续状态 / 6 codegen guards | §4.5 base rate = 1 ms 与 C4 fixed-step 一致(声明,具体 Solver 由 G2) |
| [D1 FMS 功能设计](../D-fms/D1-fms-functional.md)(reviewed 2026-05-08) | D1 §4.7 OQ4 transitional(`Control_Out_Bus` 作为 FMS 输入,Phase 2 启用)/ §4.8 OQ2 macro 保留 + leaf 重写 | §4.3 VP-Control-Out-Echo 默认 ON Phase 2 = D1 §4.7 OQ4 transitional 选项 2 |
| [D2 FMS 结构设计](../D-fms/D2-fms-structural.md)(reviewed 2026-05-08) | D2 §4.1 FMS 顶层 6 子模块 / 端口集 | §4.1 FMS block 端口集精确匹配 D2 §4.1 |
| [E1 Controller 功能设计](../E-controller/E1-controller-functional.md)(reviewed 2026-05-08) | E1 cmd_mask 裁剪规则 + 环路启用 | §4.1 wiring 中 Controller 输入 bus 集与 E1 一致(`FMS_Out_Bus` + `INS_Out_Bus`,Phase 2 顶层契约;harness-only `Plant_States_Bus` 通过 VP-Plant-States-Bypass 旁路,Controller 顶层模型本身不暴露) |
| [E2 Controller 结构设计](../E-controller/E2-controller-structural.md)(reviewed 2026-05-08) | E2 §4.1 Controller 顶层 3-bus 边界 / VP-4 harness-only Plant_States_Bus 旁路位置 | §4.3 VP-Plant-States-Bypass 是 E2 VP-4 的 harness 层物理落点 |
| [F1 ins_stub 功能设计](../F-ins-contract/F1-ins-stub-functional.md)(reviewed 2026-05-08) | F1 §4.1 ideal/noisy variant + variant_mode | §4.3 VP-INS-Variant 控制变量 = F1 + F2 锁定的 `variant_mode` |
| [F2 ins_stub 结构设计](../F-ins-contract/F2-ins-stub-structural.md)(reviewed 2026-05-08) | F2 §4.1 ins_stub 顶层 7 块 + 端口集 / §4.2 mask parameter `variant_mode` / §4.6 Option B harness-local Simulink.Parameter | §4.1 ins_stub 端口集与 §4.2.4 init 调用顺序 = F2 §4.1 / §4.2 / §4.8 投影到顶层 |

注:本工作项 `contract_impact = no`。G1 是 harness 顶层装配,不修改 firmware-visible bus / enum / parameter / EXPORT / period / `*_init` / `*_step` 任一契约面;Plant / FMS / Controller / ins_stub 子模块的契约由各自工作项与 B 区已锁定,G1 只**消费**它们各自暴露的端口集与已批准的 model-reference 边界。三份模型的 codegen 输出独立产生,与 firmware drop-in 路径(架构 v1 §5.1.10)无关 — harness 层从不进入 firmware export。FMT-Firmware 仓未挂载在本环境,涉及 firmware 头文件的间接引用走 [B5](../B-contracts/B5-ins-bus-mirror.md) mirror artifact 与各上游已落 `<pending hash>` 占位约定;G1 自身**不**新引入 firmware 引用。

## 4. 设计内容

### 4.1 顶层子系统拓扑

#### 4.1.1 顶层块清单

`model/harness/mil/mil_top.slx` 顶层共 **9 个 block + logsout 集合 + 3 个 mask parameters**(per [A5 §4.1.2](../A-architecture/A5-variant-strategy.md) Model Reference 边界):

| # | 块 | 类型 | 路径 / 实例化 | Phase 2 行为 | 速率 |
|---:|---|---|---|---|---|
| **1** | `Pilot_Cmd_Source` | Subsystem(harness-local) | 顶层 Subsystem;G4 owns 内部 | 输出 `Pilot_Cmd_Bus`(per [B1 §4.x](../B-contracts/B1-bus-inventory.md));G4 决定时间序列格式 | 20 ms(对齐 FMS;G4 owns 内部速率) |
| **2** | `GCS_Cmd_Source` | Subsystem(harness-local) | 顶层 Subsystem;Phase 2 = constant-zero | 输出 `GCS_Cmd_Bus` 全字段 ZERO(per A6 §4.4.1 默认类别)| 20 ms |
| **3** | `Auto_Cmd_Source` | Subsystem(harness-local) | 顶层 Subsystem;Phase 2 = constant-zero | 输出 `Auto_Cmd_Bus` 全字段 ZERO | 20 ms |
| **4** | `Mission_Data_Source` | Subsystem(harness-local) | 顶层 Subsystem;Phase 2 = constant-zero | 输出 `Mission_Data_Bus` 全字段 ZERO(home_lla 例外见 §4.4.4)| 20 ms |
| **5** | `Environment_Info_Source` | Subsystem(harness-local) | 顶层 Subsystem | 输出 `Environment_Info_Bus`,字段 PARAM(per A6 §4.4.2 Environment_Info_Bus 行)从 PLANT_PARAM 派生 | 1 ms(常数,可任意速率;选 1 ms 与 Plant 对齐免 RT) |
| **6** | `States_Init_Source` | Subsystem(harness-local) | 顶层 Subsystem | 输出 `States_Init_Bus`,字段 PARAM(per A6 §4.4.2 + B3 PLANT_PARAM)| 1 ms(常数;init 时一次性消费) |
| **7** | `Plant` | **Model Reference** | 引用 `model/plant/Plant.slx`(per [C2 §4.1](../C-plant/C2-plant-structural.md))| 输入:`Control_Out_Bus` + `Environment_Info_Bus` + `States_Init_Bus` + reset trigger + H4 fault input(per C2 §4.7);输出:`Plant_States_Bus` + `Extended_States_Bus` + 5 sensor buses(per C1 / C2)| 1 ms |
| **8** | `FMS` | **Model Reference** | 引用 `model/fms/mc_fms/FMS.slx`(per [D2 §4.1](../D-fms/D2-fms-structural.md))| 输入:`Pilot_Cmd_Bus` + `GCS_Cmd_Bus` + `Auto_Cmd_Bus` + `Mission_Data_Bus` + `INS_Out_Bus` + `Control_Out_Bus`(transitional per D1 §4.7);输出:`FMS_Out_Bus` | 20 ms |
| **9** | `Controller` | **Model Reference** | 引用 `model/control/mc_controller/Controller.slx`(per [E2 §4.1](../E-controller/E2-controller-structural.md))| 输入:`FMS_Out_Bus` + `INS_Out_Bus`(顶层模型契约级输入仅这两个);输出:`Control_Out_Bus`;harness-only `Plant_States_Bus` 旁路通过 VP-Plant-States-Bypass 在 Controller 顶层之**外**注入(详见 §4.3.3) | 5 ms |
| **10** | `ins_stub` | plain Subsystem | 直接装入(非 Model Reference;per [F2 §4.1](../F-ins-contract/F2-ins-stub-structural.md))| 输入:`Plant_States_Bus`(1 ms,内部 RB-01/07 downsample 到 10 ms)+ `Environment_Info_Bus` + harness-local knob set;输出:`INS_Out_Bus`(10 ms)| 10 ms |
| **11** | logsout outport 集合 | Outport / 信号路由 | G3 owns | §4.6 forward-cite;G1 保证每条契约 bus 在顶层有可记录路径 | (各自速率) |

variant 控制变量(mask parameters / model workspace):见 §4.3。

#### 4.1.2 顶层拓扑(Mermaid)

```mermaid
flowchart LR
    subgraph SRC[Sources · harness-local Subsystems]
        PC[Pilot_Cmd_Source<br/>20 ms · G4 owns]
        GC[GCS_Cmd_Source<br/>20 ms · const-zero]
        AC[Auto_Cmd_Source<br/>20 ms · const-zero]
        MD[Mission_Data_Source<br/>20 ms · const-zero]
        EI[Environment_Info_Source<br/>1 ms · PARAM]
        SI[States_Init_Source<br/>1 ms · PARAM<br/>per A6 §4.4.2]
    end

    subgraph CORE[Core models · Model Reference]
        PL[Plant<br/>= Plant.slx<br/>1 ms]
        FM[FMS<br/>= FMS.slx<br/>20 ms]
        CT[Controller<br/>= Controller.slx<br/>5 ms]
    end

    subgraph HARN[Harness fixtures]
        IS[ins_stub Subsystem<br/>10 ms · variant_mode]
        VP3{{VP-Plant-States-<br/>Bypass<br/>default OFF}}
        VP4{{VP-Control-Out-<br/>Echo<br/>default ON Phase 2}}
    end

    SI --> PL
    EI --> PL
    EI --> IS
    PC --> FM
    GC --> FM
    AC --> FM
    MD --> FM

    CT -- "Control_Out_Bus<br/>(RB-02 ZOH 5→1 ms)" --> PL
    PL -- "Plant_States_Bus<br/>(1 ms)" --> IS
    PL -- "Plant_States_Bus<br/>(harness-only via VP3)" -.-> VP3
    VP3 -.->|"Plant_States_Bus<br/>SIH/HIL only"| CT
    IS -- "INS_Out_Bus<br/>(RB-03 ZOH 10→5 ms)" --> CT
    IS -- "INS_Out_Bus<br/>(RB-04 downsample 10→20 ms)" --> FM
    FM -- "FMS_Out_Bus<br/>(RB-05 ZOH+Latch 20→5 ms)" --> CT
    CT -- "Control_Out_Bus<br/>(echo via VP4)" -.-> VP4
    VP4 -.->|"Control_Out_Bus<br/>transitional Phase 2"| FM

    PL -. logsout .-> LOG[(logsout集合<br/>G3 owns)]
    FM -. logsout .-> LOG
    CT -. logsout .-> LOG
    IS -. logsout .-> LOG
```

#### 4.1.3 Wire connections(逐边 bus / 速率 / RB-NN)

下表为 §4.1.2 拓扑图中**每条数据边**的契约级注解。bus 字段定义见 [B1](../B-contracts/B1-bus-inventory.md);速率边界策略见 [A4 §4.4](../A-architecture/A4-rate-boundaries.md);A4 边界文档化模板(A4 §4.7)在 G1 顶层装配中由本表实现。

| # | 来源 | 去向 | Bus | 速率边界 | 策略(per A4 §4.3) | 备注 |
|---:|---|---|---|---|---|---|
| W-01 | `Controller` | `Plant` | `Control_Out_Bus` | RB-02 | ZOH 5 ms → 1 ms | 慢→快;C2 §4.4.7 输出装配 |
| W-02 | `Plant` | `ins_stub` | `Plant_States_Bus` | RB-01(harness 内退化为 RB-07)| sample-time-based downsample 1 ms → 10 ms | per F2 §4.1.3 入口在 ins_stub 内部 |
| W-03 | `Plant` | (logsout / VP-3 旁路) | `Plant_States_Bus` 与 5 sensor buses | (各自速率,非 RB)| 直接连 logsout outport;VP-3 OFF 时不进入 Controller | 同时 H4 input port wiring(per C2 §4.7)预留入口 |
| W-04 | `ins_stub` | `Controller` | `INS_Out_Bus` | RB-03 | ZOH 10 ms → 5 ms | per A4 §4.4 RB-03;validity 位与字段值同步老化 |
| W-05 | `ins_stub` | `FMS` | `INS_Out_Bus` | RB-04 | sample-time-based downsample 10 ms → 20 ms | per A4 §4.4 RB-04 |
| W-06 | `FMS` | `Controller` | `FMS_Out_Bus` | RB-05 | ZOH + Latch 20 ms → 5 ms | bus 整体 latch(cmd_mask + ctrl_mode + reset + setpoint 同帧),per A4 §4.4 RB-05 + D6 |
| W-07 | `Pilot_Cmd_Source` | `FMS` | `Pilot_Cmd_Bus` | RB-06 | single-element queue + latch on FMS step start | per A4 §4.4 RB-06;G4 决定来源端节奏,latch 在 FMS 边界 |
| W-08 | `GCS_Cmd_Source` | `FMS` | `GCS_Cmd_Bus` | RB-06 | 同 W-07 | Phase 2 const-zero,但策略仍走邮箱 |
| W-09 | `Auto_Cmd_Source` | `FMS` | `Auto_Cmd_Bus` | RB-06 | 同 W-07 | Phase 2 const-zero |
| W-10 | `Mission_Data_Source` | `FMS` | `Mission_Data_Bus` | RB-06 | 同 W-07 | Phase 2 const-zero(home_lla 例外见 §4.4.4) |
| W-11 | `Environment_Info_Source` | `Plant` | `Environment_Info_Bus` | (无;1 ms 同速率) | 直连 | per C2 §4.4.2 S1 |
| W-12 | `Environment_Info_Source` | `ins_stub` | `Environment_Info_Bus`(`home_lla` / `mag_field_ned_gauss` 子集) | (无;1 ms 进 ins_stub 后内部 downsample) | 直连 | per F2 §4.3.1 字段映射 |
| W-13 | `States_Init_Source` | `Plant` | `States_Init_Bus` | (init 时一次性) | direct | per A6 §4.4.2 Plant 行 + C2 §4.4.1 S0 |
| W-14 | `Controller` | `FMS` | `Control_Out_Bus`(echo,经 VP-Control-Out-Echo) | RB(Controller 5 ms → FMS 20 ms,downsample) | sample-time-based downsample 5 ms → 20 ms | **transitional**(D1 §4.7 OQ4 选项 2);Phase 2 默认 ON;A3 §4.4.5 选项 2 已为此保留边界 |
| W-15 | `Plant` | `Controller`(经 VP-Plant-States-Bypass) | `Plant_States_Bus` | RB-01 | sample-time-based downsample 1 ms → 5 ms | **harness-only debug**(A5 §4.5 + E2 VP-4);Phase 2 默认 OFF |

约束:本表所有"策略"/"速率边界"必须严格按 [A4](../A-architecture/A4-rate-boundaries.md) 已锁定术语与策略一一对应;G1 不引入 A4 之外的 transition 模式。Solver 配置项(如何在 Simulink 中实现这些 transition 块的具体放置 / deterministic 模式 / Sample Time 着色)由 [G2](G2-rate-scheduling.md) forward-cite 闭合。

#### 4.1.4 Cmd source 子系统(harness-local)的内部规则

`GCS_Cmd_Source` / `Auto_Cmd_Source` / `Mission_Data_Source`:

- 内部仅一个 Bus Creator;字段全部 `Constant`(per A6 §4.4.1 默认 ZERO 类别);`enum` 字段取 B2 锁定的 disarmed-equiv 值。
- **不**实施任何运行期切换逻辑(不挂 H4 fault input,不与 Pilot_Cmd_Source 串联)。
- `Mission_Data_Source.home_lla` 字段例外:从 `States_Init_Source` 派生(同 PARAM 来源),保证 home 与 init 位置一致(per A6 §4.4.3 self-consistency);具体派生由 §4.4.4 init 脚本在加载 PARAM 时同步写入两个 source 的 model workspace。
- 端口形状(每 bus 一个 outport)精确符合 [B1](../B-contracts/B1-bus-inventory.md) schema;G1 不重复定义字段。

`Environment_Info_Source`:

- 内部 Bus Creator;字段从 PLANT_PARAM 派生(per [B3](../B-contracts/B3-parameter-schema.md);典型字段含 gravity / wind / mag_field / home_lla / 大气模型常数;具体字段由 [C1 §4.7](../C-plant/C1-plant-functional.md) 锁定的 `Environment_Info_Bus` 字段集 + [B1](../B-contracts/B1-bus-inventory.md) schema)。
- **不**在仿真期间随时间变化(Phase 2);风扰随时间变化的需求由 H4 故障注入实现,通过 Plant 的 H4 fault input port(C2 §4.7)注入,不污染 `Environment_Info_Source`。
- 速率选 1 ms(与 Plant 对齐免引入 RT 块);常数信号在 Simulink 中可视为任意 sample time。

`States_Init_Source`:

- 详见 §4.4.4 init wiring。

`Pilot_Cmd_Source`:

- 顶层 Subsystem,内部由 [G4](G4-pilot-injection.md) owns(forward-cite);本文件只锁端口形状(1 个 outport `Pilot_Cmd_Bus`)与 wiring(W-07)。
- §4.7 forward-cite。

### 4.2 Model Reference 策略

#### 4.2.1 决策依据

per [A5 §4.1.2](../A-architecture/A5-variant-strategy.md):**Plant / FMS / Controller 三个顶层模型采用 Model Reference 作为顶层封装机制**。理由(与 A5 一致):

1. **firmware drop-in 模式匹配**:Model Reference 是唯一直接产出独立 codegen 单元(`<Module>.c/.h`)的机制,匹配架构 v1 §5.1.10 lib/ 目录契约。Library / Subsystem 都会被宿主 inline。
2. **独立 codegen 表面与契约 diff 匹配**:[B4 §4.1](../B-contracts/B4-contract-diff.md) 24 个 Check ID 逐模块比对 EXPORT / PARAM / 符号存在性,Model Reference 的产物粒度天然对齐这一比对边界。
3. **增量 build / 加快开发循环**:Plant 改动不重 build FMS / Controller;`slprj` 缓存提供模块级隔离。Phase 2 多旋翼切片下,Controller 5 ms 热路径(per [E5](../E-controller/E5-performance-budget.md) Wave 10 sibling)的反复 codegen 成本由此显著降低。
4. **Cross-vehicle reuse**:Phase 5 扩展第二机型时,顶层 referenced model 可在 vehicle harness 间复用,不需要复制顶层装配。
5. **接口强约束契合 contract-first**:Model Reference 的 Inport / Outport 必须显式 bus 化,把契约风险挡在编辑期。

ins_stub 是 plain Subsystem(per [F2 §4.1.4](../F-ins-contract/F2-ins-stub-structural.md) + [A5 §4.1.2 决策摘要表](../A-architecture/A5-variant-strategy.md));理由:harness-only fixture,无 firmware 契约,无独立 codegen 需求,加 Model Reference 反而引入不必要的 build cache 与配置成本。

#### 4.2.2 Model Reference 实例化

每个 Model Reference 块在 G1 顶层装配中:

- **Model name** 指向其顶层 `.slx` 路径(`model/plant/Plant.slx` / `model/fms/mc_fms/FMS.slx` / `model/control/mc_controller/Controller.slx`)。具体路径常量由 [A2](../A-architecture/A2-naming-conventions.md) 命名规则锁定,G1 引用。
- **Sample Time** 设为 `-1`(Inherit;与被引用模型的 Solver 配置一致);Plant=1 ms / FMS=20 ms / Controller=5 ms 由各模型自身 Configuration Reference 锁。Solver 配置由 [G2](G2-rate-scheduling.md) forward-cite。
- **Inport / Outport 端口集**精确等同于 [C2 §4.1](../C-plant/C2-plant-structural.md) / [D2 §4.1](../D-fms/D2-fms-structural.md) / [E2 §4.1](../E-controller/E2-controller-structural.md) 锁定的端口形状。G1 不在 harness 顶层增加或删除任何端口。
- **Configuration Reference**:每个被引用模型使用独立 Configuration Reference(per [I4](../I-tooling/I4-codegen-config.md) 设计;G1 不锁,只声明三个模型必须使用各自匹配 firmware 契约的 ert.tlc 配置)。

#### 4.2.3 参数传递(Model Workspace + Data Dictionary)

被引用模型的 PARAM 来源 per [B3 §4.6](../B-contracts/B3-parameter-schema.md):

- **`PLANT_PARAM`** 注入 `Plant.slx` 的 Model Workspace;`States_Init_Source` 与 `Environment_Info_Source` 的常数字段从同一份 PLANT_PARAM 派生(详见 §4.4.4)。
- **`FMS_PARAM`** 注入 `FMS.slx` 的 Model Workspace。
- **`CONTROL_PARAM`** 注入 `Controller.slx` 的 Model Workspace(含 [E3 §4.6](../E-controller/E3-loops-algorithm.md) sealed-amendment 新增的 5 个 anti-windup 字段)。
- **harness-local 知识**(`ins_stub_knobs` per [F2 §4.6.2](../F-ins-contract/F2-ins-stub-structural.md))加载到 harness 顶层 Model Workspace(`mil_top.slx` 自身的 Model Workspace),**不**进入三份 model-reference 的 Workspace,**不**进入 PLANT_PARAM。
- 加载顺序:`FMT_Model_Init.m`([I1](../I-tooling/I1-init-script.md) owns)在 Simulink 启动前把 PARAM struct 装载到对应 Model Workspace;G1 在 PreLoadFcn 中调用 `FMT_Model_Init.m`(具体 callback 集成由 I1 设计)。

#### 4.2.4 Model Reference 的边界纪律

1. **顶层 harness 的 wiring 必须只走 Inport / Outport**;不允许 Goto / From 跨 Model Reference 边界(Simulink 限制 + 强约束所要求)。
2. **不**在 Model Reference 内部反向引用 harness 顶层信号;harness 顶层之于被引用模型是 read-only(读端口、写端口)。
3. **Model Reference 不嵌套引用其他 Model Reference 用于机型 leaf**(per A5 §4.1.2:vehicle leaf = Variant Subsystem,而非另一层 Model Reference)。Phase 2 单机型下该约束自然满足。
4. **harness-only 装配**(`Pilot_Cmd_Source` 等,以及 ins_stub)以**plain Subsystem** 形式直接放在 `mil_top.slx` 的画布上,**不**封装为额外的 Model Reference。
5. 三份 Model Reference 各自拥有独立的 codegen 输出根目录(`slprj/ert/<ModelName>/`);harness 顶层模型本身**不**用于 codegen(per A5 §4.6.4 / 架构 v1 §11.3:harness 不 export)。

### 4.3 顶层变体接入点

#### 4.3.1 接入点总览

顶层共 **5 个 variant attachment points**(VP-NN);每个有独立的控制变量与 Phase 2 默认值,彼此正交。

| ID | 接入点名 | 控制变量 | 控制变量类型 | Phase 2 默认 | 落地位置 | forward path |
|---|---|---|---|---|---|---|
| **VP-Vehicle** | Plant + FMS + Controller 多机型 | `MIL_VEHICLE`(harness-level mirror)| enum {MULTICOPTER}(per A5 §4.2.1)| `MULTICOPTER`(degenerate) | 三份 Model Reference 内部 Variant Subsystem(per A5)| Phase 5:增加 fixwing 等(per A5 §4.4.4) |
| **VP-INS-Variant** | ins_stub IDEAL vs NOISY | `INS_VARIANT_MODE` → ins_stub mask param `variant_mode`(per F2 §4.2.2) | enum {IDEAL, NOISY} | `IDEAL`(per F1 §4.4.2 / risk 4 mitigation:scenario 显式 NOISY 时切换) | ins_stub Subsystem mask | scenario YAML 注入 NOISY;H1 + H4 决定矩阵 |
| **VP-Plant-States-Bypass** | Controller 旁路 `Plant_States_Bus` | `VP_PLANT_STATES_BYPASS`(harness mask param)| boolean | `false`(OFF;per E2 VP-4 + A5 §4.5)| harness 顶层 Variant Source(在 Controller block 外)| Phase 3 SIH/HIL:用于估计器 vs truth 对比 |
| **VP-Control-Out-Echo** | FMS ← Controller `Control_Out_Bus` 回送 | `VP_CONTROL_OUT_ECHO`(harness mask param)| boolean | `true`(ON;per D1 §4.7 OQ4 transitional)| harness 顶层 Variant Source(在 W-14 wire 上)| Phase 3 评估;Phase 5 移除 |
| **VP-Harness-Variant** | MIL / SIH / HIL | `HARNESS_VARIANT`(per A5 §4.5)| enum {MIL, SIH, HIL} | `MIL`(degenerate;Phase 2 仅 MIL)| `mil_top.slx` 顶层 Variant Source(整个 harness 模型集) | Phase 3 SIH;Phase 4+ HIL |

冻结声明(per [A5 §4.2.2](../A-architecture/A5-variant-strategy.md)):上述 5 个 VP 之外,本文件**不**在 G1 顶层引入任何额外 variant 轴。Phase 2 期间任何新增第二条变体轴的提议视为策略偏离,需走 INDEX 决策日志。

#### 4.3.2 VP-Vehicle(透传至模块内部)

G1 顶层**不**再次实例化 vehicle Variant Subsystem;每份 Model Reference(`Plant.slx` / `FMS.slx` / `Controller.slx`)在自身内部承载 vehicle Variant Subsystem(per [C2 §4.4](../C-plant/C2-plant-structural.md) / [D2](../D-fms/D2-fms-structural.md) / [E2 VP-1..VP-3](../E-controller/E2-controller-structural.md) + A5 §4.1.2 decision)。

控制变量 `MIL_VEHICLE` 在 harness 顶层 Model Workspace 中定义(per A5 §4.2.1 单元素 enum {MULTICOPTER}),通过 Configuration Reference 传递到三份 Model Reference 的 Model Workspace,由各模型内部 Variant Subsystem 消费。

Phase 2 该控制变量取唯一值 `MULTICOPTER`,实际生效形式是单 active variant — 三份 Model Reference 各只生成 multicopter leaf 代码(per A5 §4.6.2 "Variant Sources / Single design")。

#### 4.3.3 VP-Plant-States-Bypass(关键 harness-only variant)

**位置**:在 `Plant` 块的 `Plant_States_Bus` 输出与 `Controller` 块的 input 之间,插入一个 **Variant Source 块**(在 Controller 顶层 Model Reference **之外**,per [A5 §4.5](../A-architecture/A5-variant-strategy.md) + [E2 §4.4 VP-4](../E-controller/E2-controller-structural.md))。

**形状**:
- Variant Source 选项 1(`VP_PLANT_STATES_BYPASS == true`):W-15 wire 激活,`Plant_States_Bus` 经 1 ms → 5 ms downsample 注入 Controller 的某个 harness-only inport(由 E2 在 SIH/HIL variant 中暴露,Phase 2 不实施)。
- Variant Source 选项 2(`VP_PLANT_STATES_BYPASS == false`,Phase 2 默认):W-15 wire 禁用,Controller 顶层模型只接 `FMS_Out_Bus` + `INS_Out_Bus`(per E2 §4.1 顶层 3-bus 边界)。

**关键纪律**:
- Controller 顶层模型(`Controller.slx`)的 codegen 输出**不**含 `Plant_States_Bus` 输入(per A5 §4.5 + E2 §4.1 — Controller 契约级输入仅 2 bus + 输出 1 bus)。VP-Plant-States-Bypass 的"开启"形态在 Phase 3 SIH/HIL 启动时由 E2 在 Controller 顶层模型内增加一个 harness-only 的 Variant 入口;Phase 2 该入口不存在,Variant Source 选项 1 在 Phase 2 永远 unreachable。
- G1 顶层在 Phase 2 仍**保留** Variant Source 块的占位(选项 1 内部为 Terminator),以便 Phase 3 启用时只需在 E2 / G1 共同评审下打开 Variant Source 的活动分支。
- H4 fault injection 的"truth-state aware"故障注入(若有)走 Plant H4 input port(C2 §4.7),**不**经 VP-Plant-States-Bypass。VP-3 严格只为"Controller 顶层调试观察 truth"服务。

#### 4.3.4 VP-Control-Out-Echo(D1 OQ4 transitional 落地)

**位置**:在 `Controller` 块的 `Control_Out_Bus` 输出 → `FMS` 块的 input 之间,插入一个 **Variant Source 块**(在 FMS Model Reference **之外**)。

**形状**:
- 选项 1(`VP_CONTROL_OUT_ECHO == true`,Phase 2 默认):W-14 wire 激活,`Control_Out_Bus` 经 5 ms → 20 ms downsample 进入 FMS 的 `Control_Out_Bus` input port(per [D2 §4.1](../D-fms/D2-fms-structural.md) FMS 顶层 6 input bus 之一,符合 D1 §4.7 OQ4 选项 2)。
- 选项 2(`VP_CONTROL_OUT_ECHO == false`):W-14 wire 禁用;FMS 的 `Control_Out_Bus` input 接 const-zero(`Control_Out_Bus` 全字段 ZERO 默认);保留 input port 不破坏契约。

**Phase 2 默认 ON 的根据**:
- D1 §4.7 OQ4 transitional 决策:Phase 2 启用 `Control_Out_Bus` 作为 FMS 输入,仅用于 MIL parity / Safety cross-check / 诊断 logging,不作为 setpoint 来源。
- 保持 firmware drop-in 兼容(架构 v1 §5.1.4 "Current FMS interface code also reads `Control_Out_Bus`")。

**Phase 5 退出**:per D1 §4.7 退出计划,Phase 5 时 D-area / I4 codegen 修订移除 FMS interface 该 input;G1 同步把 VP-Control-Out-Echo 默认翻转为 OFF 并最终删除 Variant Source。

#### 4.3.5 VP-INS-Variant(ins_stub IDEAL/NOISY 切换)

**位置**:ins_stub Subsystem 的 mask parameter `variant_mode`(per [F2 §4.2.2](../F-ins-contract/F2-ins-stub-structural.md))。

**G1 顶层暴露**:harness 顶层 Model Workspace 定义 `INS_VARIANT_MODE` 变量 → 通过 ins_stub mask 表达式 `variant_mode = INS_VARIANT_MODE` 透传。Phase 2 默认 `IDEAL`(per F1 §4.4.2 / risk 4 mitigation:H1 / H4 至少有一条 noisy 场景在 Phase 2 退出条件中要求,scenario 触发时改为 `NOISY`)。

**与其他 VP 的正交性**:与 VP-Vehicle / VP-Plant-States-Bypass / VP-Control-Out-Echo / VP-Harness-Variant 互相独立。任何组合(2^? × |MIL_VEHICLE|)在原则上都合法,但 Phase 2 实际矩阵由 [H1](../H-verification/H1-scenario-catalog.md) 锁定。

#### 4.3.6 VP-Harness-Variant(整个 harness 维度)

**位置**:`mil_top.slx` 顶层之上(若有 SIH / HIL 顶层)— Phase 2 仅 MIL,所以**不**实施;只在 §4.3.1 表中保留 forward path。

**Phase 2 落地**:`HARNESS_VARIANT = MIL`(degenerate),整个 G1 文件 = MIL 顶层;SIH / HIL 顶层模型由 Phase 3 / Phase 4 工作项创建,与 `mil_top.slx` 共享被引用的 Plant / FMS / Controller(Model Reference 的复用红利)。

**正交性**:`HARNESS_VARIANT` 与 vehicle 轴独立(per A5 §4.5 注释)。

#### 4.3.7 不在 G1 顶层引入的 variant(防过度抽象)

per A5 §4.2.2 冻结清单 + §4.3.1 容量预算(单 Variant Subsystem ≤ 2 vehicle / 嵌套 ≤ 2 / 模块内 ≤ 5):

- **不**在顶层为 RB-NN 边界引入"transition 策略 variant"(策略由 A4 锁定)。
- **不**为 logsout 启停引入 variant(G3 决定信号集与启停);若 G3 需要 variant,由 G3 自身在 logsout 子系统内实现,不污染顶层。
- **不**为 sample-time 速率(Plant 1 ms / Controller 5 ms / FMS 20 ms)引入 variant(per A4 §4.5 整数倍同相对齐 + 架构 v1 §13);任何速率改动等价于 A4 重新评审。
- **不**为 init 顺序引入 variant(A6 §4.5.1 已声明顺序不影响最终稳态)。

### 4.4 Init / Reset Wiring

#### 4.4.1 init 调用链(per A6 §4.5.1)

`mil_top.slx` 的 PreLoadFcn / InitFcn 调用 `FMT_Model_Init.m`([I1](../I-tooling/I1-init-script.md) owns;G1 只声明义务):

```text
FMT_Model_Init.m  (I1 owns)
  ├─ Load PLANT_PARAM        → Plant.slx Model Workspace
  ├─ Load FMS_PARAM          → FMS.slx Model Workspace
  ├─ Load CONTROL_PARAM      → Controller.slx Model Workspace
  ├─ Load ins_stub_knobs     → mil_top.slx Model Workspace
  ├─ Derive States_Init_Bus  ← PLANT_PARAM (per A6 §4.4.2 + B3)
  ├─ Derive Environment_Info_Bus ← PLANT_PARAM (per A6 §4.4.2 + C1 §4.7)
  └─ Derive Mission_Data_Source.home_lla ← States_Init_Bus.position  (self-consistency per A6 §4.4.3)
```

仿真启动后(Simulink 进入 step 之前),三份 Model Reference 各自调用其 `*_init`(无参,per [A6 §4.2.1](../A-architecture/A6-init-reset-contract.md));推荐顺序(per A6 §4.5.1 MIL 路径):

```text
t < 0  Simulink build & link
t = 0− init phase:
       Plant_init()       (经 plant_interface_init 等价;模型仓 Model Reference 自动调用)
       ins_stub_init()    (Stateflow chart 进入 NOT_READY,timer 归零;per F2 §4.4.3)
       FMS_init()         (FMS_Out_Bus → 安全态 disarmed-equiv;per A6 §4.4.2)
       Controller_init()  (Control_Out_Bus → 零推力;per A6 §4.4.2)
t = 0  first step hit:
       all four step at integer-multiple sample hits
       per A4 §4.5 (Plant 1 ms / ins_stub 10 ms / Controller 5 ms / FMS 20 ms 同相起步)
```

注:A6 §4.5.1 声明 init 顺序不影响最终稳态,G1 推荐顺序仅是"过渡期 ≤ 1 step 最短"的实践最优,**不**强制(防止 Simulink Model Reference 调度器决定不同顺序时引入 race)。

#### 4.4.2 各模块 init 副作用边界(per A6 §4.2.2 引用)

- `Plant_init`:把 Plant 内部刚体 / 电机 / 大气 / 积分器内部状态置 §4.4.2 类别(per [C2 §4.7](../C-plant/C2-plant-structural.md) + [C4](../C-plant/C4-numerics.md));`States_Init_Bus` 仅在第一个 step 被 Plant 消费(per [C2 §4.4.1 S0](../C-plant/C2-plant-structural.md))。
- `FMS_init`:`FMS_Out_Bus` → disarmed-equiv 安全态(per A6 §4.4.2 + [D2 §4.7](../D-fms/D2-fms-structural.md));mode = disarmed,cmd_mask = 0,reset 字段(若 B1 锁定存在)= 0。
- `Controller_init`:`Control_Out_Bus` → 零推力(per A6 §4.4.2 + [E2](../E-controller/E2-controller-structural.md));所有积分器 / anti-windup 状态归零。
- `ins_stub_init`:Stateflow chart `NOT_READY` 状态;timer 归零;noise / delay buffer 清零(per F2 §4.4.3 + §4.5);`INS_Out_Bus` 各字段为 ZERO,`INS_Status` = NOT_READY,`INS_Flag` = 0。

A6 §4.4.3 self-consistency 在 G1 顶层的体现:`Plant.States_Init_Bus.position` 与 `Mission_Data_Source.home_lla` 在 init 脚本中同源派生(§4.4.1 末行),保证 hover/home 相对位置在 init 后 byte-equal;由 H3 baseline 校验。

#### 4.4.3 Reset wiring

per [A6 §4.3.1](../A-architecture/A6-init-reset-contract.md) 五类触发源:

| 触发源 | G1 顶层入口 | 实施 |
|---|---|---|
| **TS-FMS-OUT** | FMS 内部决策 → `FMS_Out_Bus.reset` 字段(若 B1 锁定存在)| Controller 在 W-06 wire 收到 `reset=1` 后下一周期内 reset 全部环路(per A6 §4.5.2 global reset 协议;per E2 §4.4.4 reset 协议) |
| **TS-EXT-CMD** | GCS / Pilot 通过 `GCS_Cmd_Bus` / `Pilot_Cmd_Bus` 进入 FMS;由 FMS 转译为 TS-FMS-OUT(集中权威)| FMS 自身处理(D2 / D3);G1 顶层不另设入口 |
| **TS-MODE-XCHG** | FMS Mode Manager 内部决策(D3)→ `FMS_Out_Bus.cmd_mask` / `ctrl_mode` 跳变 | Controller 在 W-06 wire 检测到跳变后做局部 reset(per A6 §4.5.2 local reset);G1 顶层透明转发 |
| **TS-FAILSAFE** | FMS Safety Monitor 决策(D2)→ `FMS_Out_Bus.reset=1` + cmd_mask=0 | 同 TS-FMS-OUT 路径 |
| **TS-MIL-HARNESS** | **G1 顶层一个独立 reset trigger 输入端口**(harness-only) | 路由到:(a) Plant Reset Port(per [C2 §4.6.3](../C-plant/C2-plant-structural.md));(b) ins_stub Stateflow chart reset(per [F2 §4.4.3](../F-ins-contract/F2-ins-stub-structural.md));**不**路由到 FMS / Controller(它们走 firmware-equivalent 路径,per A6 §4.5.2 关键约束 1 + §4.7.2 对称性) |

**TS-MIL-HARNESS reset trigger 端口**:

- 在 `mil_top.slx` 顶层暴露 1 个 boolean Inport(端口名 `harness_reset_trigger`),默认 disconnected = 永远 false(per C2 §4.6.3)。
- scenario 脚本 / G4 / H1 场景目录可在仿真期间通过该端口注入 reset(用于场景拼接 / 故障恢复测试)。
- 必须对齐到 sample boundary(per A6 §4.3.3),由 G2 在 Solver 配置中保证 deterministic 触发时刻。
- 该端口在 codegen 输出中**必须**被消除(per A6 §4.7.1 + I4 codegen 配置 + B4 contract diff:`harness_reset_trigger` 不出现在 firmware 路径)。Phase 2 codegen 仅作用于 Plant / FMS / Controller(harness 不 export,per A5 §4.6.4),所以该端口自然不进入 firmware。

#### 4.4.4 `States_Init_Source` 详细

per [A6 §4.4.2 Plant 行](../A-architecture/A6-init-reset-contract.md):刚体位置 / 速度 / 姿态 / 角速度的初值类别为 PARAM,来源为 PLANT_PARAM 中相应字段(具体字段名由 [B3](../B-contracts/B3-parameter-schema.md) + [C3](../C-plant/C3-multicopter-leaf.md) 锁,G1 不重述)。

`States_Init_Source` 内部:

- 仅 1 个 Bus Creator;字段全部为 `Constant` 块,值由 init 脚本(§4.4.1)从 PLANT_PARAM 派生写入 `States_Init_Source` 的 Model Workspace。
- speed / ang_rate / attitude offset 等 Phase 2 多旋翼默认 = ZERO(在地面静止);position 默认为 home 起飞点(由 PLANT_PARAM 提供 home offset)。
- 非 init 期间(t > 0)Plant 不再消费 `States_Init_Bus`(per C2 §4.4.1 S0);所以该 Source 信号在仿真期间是常数。

Reset 时(TS-MIL-HARNESS):Plant Reset Port 触发,Plant S0 重新读取 `States_Init_Bus` 等价 init(per C2 §4.6.3);G1 顶层无需重新加载脚本。

self-consistency(per A6 §4.4.3):`States_Init_Bus.position` 与 `Mission_Data_Source.home_lla` 在 §4.4.1 init 脚本中同源派生,保证一致。

### 4.5 Solver 与 rate-transition placement(forward-cite G2)

#### 4.5.1 G1 锁定的速率约束(传递给 G2)

- **base rate = 1 ms**(per [A4 §4.5](../A-architecture/A4-rate-boundaries.md) 整数倍同相对齐 + [C4](../C-plant/C4-numerics.md) Plant fixed-step 1 ms)。
- 顶层四个独立 sample time:1 ms(Plant + Environment_Info_Source + States_Init_Source)/ 5 ms(Controller)/ 10 ms(ins_stub 内部)/ 20 ms(FMS + 4 个 Cmd_Source)。
- 整数倍同相起步(per A4 §4.5):t = 0 时所有 sample hit 重合;之后各模块在自己的整数倍时刻执行。

#### 4.5.2 rate-transition 块 placement 声明

per A4 §4.4 + 本文件 §4.1.3 W-NN 表,顶层共 7 处 rate transition 位置(W-01 / W-02 / W-04 / W-05 / W-06 / W-07..W-10 同类合并 / W-14 / W-15):

| Wire | 边界 | 策略 | placement |
|---|---|---|---|
| W-01 | Controller 5 ms → Plant 1 ms | RB-02 ZOH | Plant block 的 `Control_Out_Bus` 输入端;由 Simulink 自动推断 |
| W-02 | Plant 1 ms → ins_stub 10 ms | RB-01 / RB-07 downsample | ins_stub 内部第 1 块(Rate Transition,per F2 §4.1.1)— **G1 顶层不放置 RT 块**,在 ins_stub 内部 |
| W-04 | ins_stub 10 ms → Controller 5 ms | RB-03 ZOH | Controller block 的 `INS_Out_Bus` 输入端 |
| W-05 | ins_stub 10 ms → FMS 20 ms | RB-04 downsample | FMS block 的 `INS_Out_Bus` 输入端 |
| W-06 | FMS 20 ms → Controller 5 ms | RB-05 ZOH + Latch(整 bus 同帧)| Controller block 的 `FMS_Out_Bus` 输入端;**关键:必须保证 bus 整体 latch**(per A4 RB-05 + D6) |
| W-07..W-10 | 外部源 (异步) → FMS 20 ms | RB-06 single-element queue + latch | FMS block 各 cmd input 端 |
| W-14 | Controller 5 ms → FMS 20 ms(VP-Control-Out-Echo) | downsample 5 → 20 | FMS block 的 `Control_Out_Bus` 输入端(VP 经过的 Variant Source 后) |
| W-15 | Plant 1 ms → Controller 5 ms(VP-Plant-States-Bypass)| RB-01 downsample | Controller block 的 harness-only `Plant_States_Bus` 输入(Phase 3+ 才存在;Phase 2 不放置) |

#### 4.5.3 G2 forward-cite

[G2](G2-rate-scheduling.md)(Wave 10 sibling)owns 全部 Solver pane 决策(fixed-step / 离散积分器选项 / Sample Time 着色检查清单 / 调度图 / RT 块 deterministic 模式);G1 仅声明 placement 与速率,具体配置项交 G2。

任何在 G2 设计中发现需要修改 G1 §4.5.2 placement 表的反馈,走 §8 变更日志 + INDEX 决策日志。

### 4.6 Logging(forward-cite G3)

#### 4.6.1 G1 在顶层提供的 logsout 装配规则

- **每条契约 bus** 在顶层至少有一个 logsout outport(`Plant_States_Bus` / `Extended_States_Bus` / 5 sensor buses / `INS_Out_Bus` / `FMS_Out_Bus` / `Control_Out_Bus` / 4 Cmd buses / `Environment_Info_Bus` / `States_Init_Bus`)。具体信号集(全 bus log vs 选择性 log)由 [G3](G3-logging.md) 决定。
- **顶层 logsout 集合块**位于 `mil_top.slx` 画布右侧,统一汇集到一个或多个 To Workspace / Outport(具体由 G3)。
- G1 不指定信号命名 / 单位 metadata / 采样时戳规则 / 可视化目录 / manifest;这些是 G3 退出条件。

#### 4.6.2 G3 forward-cite

[G3](G3-logging.md)(Wave 10 sibling)owns logsout 最小信号集(架构 v1 §13 各模块代表性信号 + INS validity + cmd_mask + mode/state)、命名 / 单位 / 时戳 / 可视化目录 / manifest。

G1 承诺:G3 锁定信号集后,G1 顶层布线要保证每条要求 log 的信号在顶层可达(无 Goto / From 跨 Model Reference 边界限制下,Model Reference 的 Outport 必须在被引用模型内已被 hook 出来 — 这是对 C2 / D2 / E2 / F2 的 backward compatibility 要求,各模块结构设计已为此预留 logsout 友好的 Outport 集)。

### 4.7 Pilot injection(forward-cite G4)

#### 4.7.1 G1 顶层提供的接入点

- **`Pilot_Cmd_Source` Subsystem**:位于 `mil_top.slx` 顶层,1 个 Outport(`Pilot_Cmd_Bus`,per B1)。
- 内部由 [G4](G4-pilot-injection.md) owns。
- W-07 wire 接到 FMS 的 `Pilot_Cmd_Bus` input,边界策略 = RB-06(per §4.1.3)。
- **可选**:scenario JSON / DSL parser(per G4 设计)— G1 不锁;G4 可选择内嵌 From Workspace 时间序列、Lookup Table 时序、或独立的 .m parser block。

#### 4.7.2 与 TS-MIL-HARNESS reset 的协调

per [A6 §4.3.3](../A-architecture/A6-init-reset-contract.md) reset 必须对齐 sample boundary;G4 时间序列在 reset 前后的连续性(reset 是否清空 Pilot 队列?是否回到时间序列 t=0?)由 G4 owns;G1 顶层只保证 reset trigger 与 Pilot_Cmd_Source 的接口不互相阻塞(`Pilot_Cmd_Source` 不消费 `harness_reset_trigger`,只是被 FMS 在自己的边界 latch)。

### 4.8 跨工作项交叉引用与 Wave 10 sibling 协调

#### 4.8.1 Wave 10 兄弟(non-blocking parallel drafting)

| 兄弟 | 协调点 | G1 承担 / 承诺 |
|---|---|---|
| [G2 多速率调度](G2-rate-scheduling.md) | §4.5 placement 表 + base rate 1 ms + 整数倍同相 | G1 锁 placement 与速率;G2 在该约束下决定 Solver 配置 |
| [G3 日志与可观测性](G3-logging.md) | §4.6 顶层每条契约 bus 都有 logsout 路径 | G1 保证 wiring 不阻塞 G3 信号选取 |
| [G4 Pilot_Cmd 注入](G4-pilot-injection.md) | §4.7 `Pilot_Cmd_Source` Subsystem 顶层接入 | G1 保留 1 个 Outport 形状,G4 owns 内部 |
| [E5 Controller 性能预算](../E-controller/E5-performance-budget.md) | §4.5 Controller 5 ms 边界(W-04 RB-03 / W-06 RB-05 / W-01 RB-02)| G1 锁边界策略与 placement;E5 在 5 ms 内做 µs 子预算 |

Wave 10 内若 G2 / G3 / G4 / E5 出现需要 G1 调整顶层结构的反馈,走 §8 变更日志(Wave 10 batch 同期评审协调)。

#### 4.8.2 受 G1 影响的下游消费者

- [I1 init 脚本](../I-tooling/I1-init-script.md):G1 §4.4.1 调用义务 → I1 设计 `FMT_Model_Init.m` 必须满足该执行链;I1 worked example 应包含 `mil_top.slx` 上下文。
- [I5 sim runner](../I-tooling/I5-sim-runner.md):G1 顶层是 sim runner 的 entry model;I5 设计的 single-run / batch / regression 三模式 CLI 都以 `mil_top.slx` 为目标。
- [H1 验证场景目录](../H-verification/H1-scenario-catalog.md):G1 §4.3 5 VPs + §4.4.3 TS-MIL-HARNESS reset trigger 是 H1 场景描述的接入点。
- [H4 故障注入目录](../H-verification/H4-fault-catalog.md):G1 §4.3.3 提到 Plant H4 input port 在顶层的 wiring(由 C2 §4.7 暴露,G1 透传);H4 场景描述这些注入点。

#### 4.8.3 与上游契约的对齐(无新增契约影响)

G1 的所有 wiring / port shape 严格沿用上游已锁定的 bus / enum / param 契约;本文件无新增 firmware-visible 修改,因此 `contract_impact = no`。

具体核对:

- 所有 bus 名引用 [B1](../B-contracts/B1-bus-inventory.md);未新增字段。
- 所有 enum 引用 [B2](../B-contracts/B2-enum-inventory.md);未新增数值。
- 所有 param 引用 [B3](../B-contracts/B3-parameter-schema.md);未新增字段。
- 所有 init 函数引用 [A6 §4.2.1](../A-architecture/A6-init-reset-contract.md) 锁定的 `void(void)` 签名;未修改。
- 所有 reset 触发源走 [A6 §4.3.1](../A-architecture/A6-init-reset-contract.md) 五类;未新增。
- 所有跨速率边界走 [A4 §4.4](../A-architecture/A4-rate-boundaries.md) RB-NN;未引入新边界 ID(本文件 §4.1.3 W-NN 是 wire-level 注解,引用现有 RB-NN)。
- harness 顶层不进入 firmware export(per A5 §4.6.4 + 架构 v1 §11.3);TS-MIL-HARNESS reset 端口在 codegen 中天然被消除(harness 不参与 codegen)。

## 5. 已知风险与悬而未决问题

- **`FMS_Out_Bus.reset` 字段是否存在于 B1 schema**
  - 影响:§4.4.3 TS-FMS-OUT 与 TS-FAILSAFE 触发依赖该字段。若 B1 锁定后发现不存在(per A6 §5 第 2 项 open),global reset 信道改用 cmd_mask + ctrl_mode 派生(per A6 §5)。G1 顶层无需修改 wiring(W-06 仍 latch 整 bus),只需在 Controller block 内部检测策略变化。
  - 处置:跟随 A6 §5 第 2 项 open;若 B1 后续修订确认无 reset 字段,本文件 §4.4.3 相应行走 §8 变更日志。

- **VP-Plant-States-Bypass 在 Phase 3 SIH/HIL 启用时,Controller 顶层模型需要新增 harness-only inport**
  - 影响:Controller 顶层模型契约级输入仅 2 bus(per E2 §4.1);VP-3 启用要求 Controller 暴露第 3 个 inport,但仅在 SIH/HIL variant 下生效(Phase 2 不存在)。
  - 处置:Phase 3 SIH 启动时由 E2 + G1 共同评审决定 Controller 顶层 SIH variant 的 inport 暴露策略;Phase 2 G1 顶层占位 Variant Source 已为此预留(选项 1 内部为 Terminator)。

- **VP-Control-Out-Echo Phase 5 退出涉及 FMS interface 函数签名修改**
  - 影响:per D1 §4.7 Phase 5 退出计划,`Control_Out_Bus` 移除是 firmware-visible 修改;G1 同步翻转 VP 默认 OFF + 删除 Variant Source。
  - 处置:Phase 5 工作项触发时由 D-area / I4 / G1 同期 INDEX 决策日志登记;Phase 2 不动作。

- **TS-MIL-HARNESS reset 端口在 codegen 中的 elimination 由 I4 codegen 配置保证,不由 G1**
  - 影响:harness 不参与 codegen(per A5 §4.6.4),`harness_reset_trigger` 自然不进入 firmware;但若未来某变体需要 harness 输出 codegen(例如 SIH 自动化场景),该端口必须被显式消除。
  - 处置:由 [I4](../I-tooling/I4-codegen-config.md) 锁定 codegen 配置时确保;本文件不重述。

- **三份 Model Reference 的 Configuration Reference 是否共享**
  - 影响:per A5 §4.1.2 / §4.6.2 推荐每模块独立 Configuration Reference 以匹配 firmware drop-in;但若 I4 锁定 vehicle leaf 之间需要某些共享配置项(例如同一 ert.tlc 基础配置),G1 顶层是否需要参与不明。
  - 处置:推迟到 I4(Wave 12);I4 退出条件已含 ert.tlc 完整配置项清单 + per-module 差异。G1 §4.2.2 仅声明三模型各自独立,具体配置由 I4。

- **`Mission_Data_Source.home_lla` 自洽性**
  - 影响:§4.4.4 要求 home_lla 与 States_Init_Bus.position 同源派生;若 D5 [多旋翼 leaf](../D-fms/D5-multicopter-leaf.md) 决定 home 字段在 FMS 中由 mode 切换时刻 latch(而非 init 时常驻),会与 G1 §4.4.4 冲突。
  - 处置:已确认 D5 reviewed;若后续 D5 修订改变 home 来源,本文件 §4.4.4 走 §8 变更日志。

- **Wave 10 sibling drafting 期间反向影响**
  - 影响:G2 / G3 / G4 / E5 在 parallel drafting 中可能反馈结构调整需求(例如 G3 要求顶层暴露额外 logsout 信号路径,或 E5 发现某 RT 块 placement 引入超预算延迟)。
  - 处置:Wave 10 batch 同期评审协调;非 co-seal 的兄弟反馈走 §8 变更日志,非阻塞。

- **TS-MIL-HARNESS reset 在 ins_stub 中的语义**
  - 影响:per F2 §4.4.3 ins_stub Stateflow chart 响应 TS-MIL-HARNESS;但 RNG sub-seed 在 reset 时是否重新派生(F2 §4.5)未在本文件锁。
  - 处置:F2 已 reviewed,内部决策由 F2 闭合;G1 顶层只把 `harness_reset_trigger` 路由到 ins_stub 的 reset 入口,不干预 chart / RNG 行为。

- **HARNESS_VARIANT 在 Phase 2 degenerate,但 Variant Source 块的 Phase 5 启用接入点未实现**
  - 影响:per A5 §4.5 / §5 第 3 项 open 与本文件 §4.3.6,Phase 2 仅 MIL,SIH/HIL 不实施;但顶层是否预留 Variant Source 块占位以便 Phase 3 启用?
  - 处置:与 VP-Plant-States-Bypass 同期(均 Phase 3 SIH 启动时落地);Phase 2 不预留,以避免占位空 Variant 在 codegen 中误生效;Phase 3 工作项创建时新增 Variant Source。

## 6. 退出条件复核

G1 在 [`00-design-plan.md §4.G`](../00-design-plan.md) 中的退出条件原文:

> 顶层 .slx 的子系统拓扑、引用模型策略、变体接入点

逐条复核:

| # | 退出条件子项 | 本文档依据 | 状态 |
|---|---|---|---|
| 1 | "顶层 .slx 的子系统拓扑" | §4.1 全节(§4.1.1 顶层块清单 9+1+3 项 / §4.1.2 Mermaid 拓扑图 / §4.1.3 W-01..W-15 wire connections 表 / §4.1.4 cmd source 子系统内部规则) | 满足 |
| 2 | "引用模型策略" | §4.2 全节(§4.2.1 三模块 = Model Reference 决策依据 5 条 / §4.2.2 实例化 / §4.2.3 参数传递 / §4.2.4 边界纪律 5 条) | 满足 |
| 3 | "变体接入点" | §4.3 全节(§4.3.1 5 VPs 总览表 / §4.3.2..§4.3.6 各 VP 详细 / §4.3.7 不引入额外 variant 防过度抽象) | 满足 |

附:Wave 10 sibling forward-cite 完整性(隐含):

| # | 隐含条件 | 本文档依据 | 状态 |
|---|---|---|---|
| 4 | G2 forward-cite | §4.5 全节(§4.5.1 速率约束 / §4.5.2 placement / §4.5.3 G2 forward-cite) | 满足 |
| 5 | G3 forward-cite | §4.6 全节(§4.6.1 顶层装配规则 / §4.6.2 G3 forward-cite) | 满足 |
| 6 | G4 forward-cite | §4.7 全节(§4.7.1 接入点 / §4.7.2 reset 协调) | 满足 |
| 7 | E5 forward-cite | §4.5.2 W-04 / W-06 / W-01 边界声明传递给 E5;§4.8.1 表行 | 满足(forward-cite) |

附:init / reset 完整性(隐含,per A6):

| # | 隐含条件 | 本文档依据 | 状态 |
|---|---|---|---|
| 8 | init 调用链 | §4.4.1 + §4.4.2 | 满足 |
| 9 | reset 触发源 5 类映射 | §4.4.3 表 | 满足 |
| 10 | self-consistency | §4.4.4(home_lla 同源派生)+ §4.4.2 末段 | 满足 |

退出条件全部满足;状态可由 reviewer 升级到 reviewed。

## 7. 下游影响

按 [`01-design-relationships.md §4.7 / §4.9`](../01-design-relationships.md):

```text
C2, D2, E2, F2 ⇢ G1
G1 → G3
G1 → G4
G1, H1, H3 ⇢ I5
```

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [G2 多速率调度](G2-rate-scheduling.md)(Wave 10 sibling) | (sibling)| §4.5.1 base rate 1 ms + 整数倍同相;§4.5.2 RT placement 表(7 处)— G2 锁 Solver 配置时引用 |
| [G3 日志与可观测性](G3-logging.md)(Wave 10 sibling)| → | §4.6 每条契约 bus 顶层可达;§4.1.3 W-01..W-15 wire 集 — G3 选取信号集时引用 |
| [G4 Pilot_Cmd 注入](G4-pilot-injection.md)(Wave 10 sibling)| → | §4.7.1 `Pilot_Cmd_Source` Subsystem 顶层接入(1 outport `Pilot_Cmd_Bus`);§4.7.2 reset 协调 |
| [E5 Controller 性能预算](../E-controller/E5-performance-budget.md)(Wave 10 sibling)| ⇢ | §4.5.2 Controller 边界(W-01 RB-02 / W-04 RB-03 / W-06 RB-05);A4 §4.6 闭环延迟上界 ~17 ms 由 G1 顶层 wiring 实现 |
| [I1 仓库初始化脚本](../I-tooling/I1-init-script.md)(Wave 12)| ⇢ | §4.4.1 调用链(`FMT_Model_Init.m` 必须执行 7 步);§4.4.4 self-consistency 派生义务 — I1 worked example 围绕该执行链 |
| [I5 仿真运行 / 批量回归脚本](../I-tooling/I5-sim-runner.md)(Wave 12)| ⇢ | §4.1 顶层模型 = I5 entry model;§4.3 5 VPs = I5 batch 配置矩阵;§4.4.3 TS-MIL-HARNESS reset trigger 端口;§4.7 scenario / Pilot 注入 |
| [H1 验证场景目录](../H-verification/H1-scenario-catalog.md)(Wave 11)| ⇢ | §4.3 5 VPs(场景级 vehicle / INS variant / VP toggle 矩阵);§4.4.3 TS-MIL-HARNESS reset 端口;§4.7 Pilot 注入 |
| [H3 回归基线策略](../H-verification/H3-regression-baseline.md)(Wave 11)| ⇢ | §4.4.4 self-consistency(byte-equal baseline 前提);§4.4.1 init 调用链(H3 baseline mat 中固化的 PARAM 与 init seed) |
| [H4 故障注入目录](../H-verification/H4-fault-catalog.md)(Wave 11)| ⇢ | §4.4.3 TS-MIL-HARNESS reset(故障恢复路径);§4.1.3 Plant H4 input port 在顶层的存在(由 C2 §4.7 透传) |

间接影响:

- [B4 契约 diff 策略](../B-contracts/B4-contract-diff.md):G1 顶层不参与 codegen,但 §4.4.3 TS-MIL-HARNESS reset 端口必须不出现在 firmware 路径(per B4 §4.1 e/f 类 Check)— 由 harness 不 export 的天然边界保证。
- [I4 Codegen 配置](../I-tooling/I4-codegen-config.md):G1 §4.2.2 三模型独立 Configuration Reference 是 I4 设计依据;Phase 5 VP-Control-Out-Echo 退出涉及 I4 修订。

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-10 | G1 author | 初稿(Wave 10;forward-cite G2 / G3 / G4 / E5 sibling drafting) |

## Self-check

- [x] frontmatter 完整,字段值合法
- [x] 上游文档全部存在且 status ≥ reviewed:架构 v1 / A1..A8 / B1..B3 / C1 / C2 / C4 / D1 / D2 / E1 / E2 / F1 / F2 全部 reviewed=pass(per [INDEX](../INDEX.md))。Wave 10 sibling(G2 / G3 / G4 / E5)作为 forward-cite 不属于 upstream(non-blocking parallel drafting,per task brief)— G1 §3 仅列已 reviewed 上游
- [x] 退出条件逐条复核完成,每条均给出依据(§6 表 3 项明确 + 4 项 sibling forward-cite + 3 项 init/reset 隐含)
- [x] 引用路径全部可点击访问(架构 v1 / A1..A8 / B1..B3 / C1..C4 / D1..D2 / E1..E2 / F1..F2 / G2..G4 / E5 / I1 / I4 / I5 / H1..H4 / B4 均使用相对路径)
- [x] 不存在 RULES §5 禁则中的内容:无 `.slx` 截图(用 Mermaid 文本拓扑);无可执行 `.m` 代码(`FMT_Model_Init.m` 调用链以伪代码块展示);无 firmware 实现重述(只引用 firmware-visible 符号 `*_init` 来自 A6,不重述实现);无 PR / branch 名;不重复 bus / enum / param 字段表(全部引用 B 区);依赖在 §3 列;`<pending hash>` 占位由各上游 A3/A6/A7/B1..B5/F1/F2 同 batch 约定承接,G1 自身不新引入 firmware 引用
- [x] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记(本文件 contract_impact=**no**:G1 是 harness 顶层装配,不修改 firmware-visible bus / enum / parameter / EXPORT / period / `*_init` / `*_step` 任一契约面;harness 不 export per A5 §4.6.4 + 架构 v1 §11.3。N/A)
- [x] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径(本文件不直接镜像 firmware 契约;所有 firmware-related 引用走上游 A6 / B1 等已落 `<pending hash>` 占位约定。N/A)
- [x] 下游影响已沿关系图识别完毕(§7 表覆盖 G2 / G3 / G4 / E5 sibling + I1 / I5 / H1 / H3 / H4 直接下游 + B4 / I4 间接下游)
- [x] 文档不超出本工作项范围:Plant / FMS / Controller / ins_stub 内部结构由 C2 / D2 / E2 / F2 已 reviewed,本文件不重述;Solver 配置交 G2;logsout 信号集交 G3;Pilot 时间序列交 G4;Controller 5 ms 子预算交 E5;PARAM 数值交 B3 + leaf;ins_stub 内部 RNG / knob 交 F2;init 脚本 IO 交 I1;sim runner CLI 交 I5;场景目录交 H1;故障注入交 H4;H3 baseline 交 H3;codegen 配置交 I4
