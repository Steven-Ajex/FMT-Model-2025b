# FMT-Model-2025b 设计阶段工作计划

最后更新:2026-05-05
状态:设计阶段已启动,实现阶段未启动

## 1. 目的

在编写任何 Simulink 模型、脚本、或生成代码之前,**完成全面的设计**,覆盖架构细化、契约、各模块的功能与结构设计、Harness、验证策略、工具链。所有设计落盘为 `docs/design/` 下的 markdown 文档。

设计阶段的退出条件:本计划中**所有工作项**达到各自"退出条件";`INDEX.md` 的"已完成设计文档清单"反映全部完成项。

## 2. 输入文档

- [架构 v1](../architecture/2026-05-05-fmt-model-architecture-v1.md) — 模块拆分、契约、阶段路线
- FMT-Firmware 现有头文件:`plant_interface.h`、`fms_interface.h`、`control_interface.h`、`ins_interface.h` 及其 `*_types.h`
- 项目 skills:`fms-architect`、`controller-architect`、`model-perf-tune`、`fmt-bus-sync`

## 3. 设计原则(适用于所有工作项)

1. **可追溯**:每个设计文档顶部声明其依赖的上游文档。
2. **可验证**:每个工作项有具体的"退出条件",非主观判断。
3. **不写实现**:设计文档只描述"是什么/为什么",不含 `.slx` 截图或 `.m` 实现代码;最多包含算法伪代码、信号清单、状态机草图(文本/Mermaid)。
4. **契约不可变**:任何动到 firmware 可见 bus/enum/参数结构的设计需独立标注,作为契约风险事项。
5. **单一来源**:bus/enum 字段在 B 区设计后,其他文档只引用,不重复定义。

## 4. 工作项总览

工作项按字母 + 序号编排,字母对应设计领域。每个工作项独立可工作、独立有退出条件。完整依赖关系见 [01-design-relationships.md](01-design-relationships.md)。

### A. 架构细化(对架构 v1 的细化)

| ID | 名称 | 产出 | 退出条件 |
|---|---|---|---|
| A1 | 架构 v1 评审与缺口闭合 | `A-architecture/A1-v1-review.md` | 列出 v1 中未细化的所有问题点;每条标注由哪个工作项闭合 |
| A2 | 命名与目录约定设计 | `A-architecture/A2-naming-conventions.md` | 文件名、库块名、bus 名、enum 名、参数 struct 字段名的命名规则全部定稿;**单位与坐标系约定(SI 单位、NED/ENU/body 等坐标系记法、角度 rad vs deg)写入命名规则** |
| A3 | 模块边界信号清单细化 | `A-architecture/A3-module-boundaries.md` | Plant/FMS/Controller 三个模块的输入/输出 bus 字段级清单与方向定稿;**每个字段附单位与坐标系标注** |
| A4 | 跨速率边界设计 | `A-architecture/A4-rate-boundaries.md` | 1ms/5ms/10ms/20ms 之间的 rate transition 位置、策略(zero-order hold / latch / queue)定稿 |
| A5 | 变体策略设计 | `A-architecture/A5-variant-strategy.md` | 多机型(Multi/FW/VTOL/...)的 Variant Subsystem vs 独立模型 抉择 + 多旋翼 leaf 的变体管理方案;**(a) 各顶层模型的 Model-Reference vs Library vs Subsystem 选型决定;(b) Phase 2 变体轴枚举 + 其他轴显式冻结;(c) 过度抽象预算上限(单 Variant 块最多支持的 vehicle 数 / leaf 文件数);(d) 记录架构 v1 §17 OQ1(vehicle scope)的最终选择** |
| A6 | Init/Reset 状态契约 | `A-architecture/A6-init-reset-contract.md` | `*_init` 行为定义、reset 触发源与时机、初值来源(参数 vs hardcoded)、reset 后的 bus 字段稳态值、Plant/FMS/Controller 在重启场景下的协议 |
| A7 | 时间与时间戳约定 | `A-architecture/A7-time-conventions.md` | `fms_interface_step(uint32_t timestamp)` 的 timestamp 单位 / epoch / 回卷处理 / 单调性约束 / dt 来源(由 timestamp 派生 vs 固定步长);Plant/FMS/Controller 之间时间戳一致性策略;模板 `template_fms` 当前签名 `(void)` 与正式签名差异的处置(契约风险) |
| A8 | 共享库块清单与归属 | `A-architecture/A8-shared-library-roster.md` | 跨 C2/D2/E2 的共享库块台账(块名、所有者、消费者、版本字段);A1 已识别的归属空白点全部闭合 |

### B. 契约设计(bus / enum / parameter)

| ID | 名称 | 产出 | 退出条件 |
|---|---|---|---|
| B1 | Bus 清单与 schema 设计 | `B-contracts/B1-bus-inventory.md` | 闭环必需 + 完整契约的全部 bus 列表;每个 bus 字段级 schema 与 firmware `*_types.h` 字段顺序、字节布局一致;**包含的覆盖范围已与 B4 退出条件呼应** |
| B2 | Enum 清单与数值锁定 | `B-contracts/B2-enum-inventory.md` | 所有契约 enum 名 + 数值 + 含义,与 firmware 一致 |
| B3 | Parameter schema 设计 | `B-contracts/B3-parameter-schema.md` | `FMS_PARAM` / `CONTROL_PARAM` / `PLANT_PARAM` 字段级定义 + 多旋翼默认值表;**每个参数显式分类为 runtime-tunable 或 compile-inlined,并给出依据** |
| B4 | 契约 diff 策略设计 | `B-contracts/B4-contract-diff.md` | diff 工具(由 I3 实现)的输入/输出格式、运行时机、失败响应规则;**必须覆盖:(a) 所有 bus 字段顺序与类型字节相等;(b) 所有 enum 数值精确相等;(c) `FMS_PARAM`/`CONTROL_PARAM`/`PLANT_PARAM` 字段顺序与类型相等;(d) `FMS_EXPORT`/`CONTROL_EXPORT`/`PLANT_EXPORT` 字段顺序、`period` 数值、`model_info[]` 长度相等;(e) 符号存在性检查 — `FMS_init`/`FMS_step`/`Controller_init`/`Controller_step`/`Plant_init`/`Plant_step` 必须出现在生成产物中**;明确 pass/fail 判定阈值 |
| B5 | INS_Out_Bus 镜像同步流程 | `B-contracts/B5-ins-bus-mirror.md` | **(a) 声明 canonical 来源(firmware 单向 mirror vs 共享 schema 子模块,记录架构 v1 §17 OQ6 的最终选择);(b)** 抽取脚本输入输出、版本标记策略;**(c) 镜像产物必须记录 firmware commit hash + 文件相对路径(RULES §5)** |

### C. Plant 设计

| ID | 名称 | 产出 | 退出条件 |
|---|---|---|---|
| C1 | Plant 功能设计 | `C-plant/C1-plant-functional.md` | 仿真物理范围(刚体/气动/电机/传感器/扰动)与各项物理保真度等级 |
| C2 | Plant 结构设计 | `C-plant/C2-plant-structural.md` | 子系统层次、共享库块清单、leaf 与 shared 边界 |
| C3 | Plant 多旋翼 leaf 设计 | `C-plant/C3-multicopter-leaf.md` | 几何、质量惯量、电机模型、分配矩阵、初始条件参数化方案;**参数名与 firmware `PLANT_PARAM` 字段名兼容性显式核对** |
| C4 | Plant 数值与积分设计 | `C-plant/C4-numerics.md` | 积分器选择、步长、初值、reset 行为、数值稳定性边界 |

### D. FMS 设计

| ID | 名称 | 产出 | 退出条件 |
|---|---|---|---|
| D1 | FMS 功能设计 | `D-fms/D1-fms-functional.md` | mode 集合、命令源、整形策略、安全触发条件、mission 范围(仅闭环切片需要的子集);**记录架构 v1 §17 OQ4(`Control_Out_Bus` 是否长期作为 FMS 输入)的决策**;**记录 OQ2(保留现有 FMS 结构 vs 重新撰写)的决策** |
| D2 | FMS 结构设计 | `D-fms/D2-fms-structural.md` | 6 大子模块(Source Selector/Mode Mgr/Safety/Mission/Shaper/Output Assembler)的边界与内部信号 |
| D3 | FMS Mode Manager 详细设计 | `D-fms/D3-mode-manager.md` | Stateflow 状态/转换/事件/守卫定稿;状态与 `VehicleStatus`/`VehicleState`/`ctrl_mode` 映射表 |
| D4 | FMS Command Shaper 详细设计 | `D-fms/D4-command-shaper.md` | 各 mode 下输出 cmd_mask + setpoint 映射;rate/jerk 限制策略 |
| D5 | FMS 多旋翼 leaf 设计 | `D-fms/D5-multicopter-leaf.md` | 多旋翼特定的 mode-cmd 映射、起飞/降落 profile;**参数名与 firmware `FMS_PARAM` 字段名兼容性显式核对** |
| D6 | FMS↔Controller 接口约定 | `D-fms/D6-fms-controller-interface.md` | `cmd_mask` 位语义、参考字段优先级、互斥规则;此文件被 D 与 E 共同遵循 |

### E. Controller 设计

| ID | 名称 | 产出 | 退出条件 |
|---|---|---|---|
| E1 | Controller 功能设计 | `E-controller/E1-controller-functional.md` | 启用的环路、cmd_mask 触发的环路裁剪规则、各环职责。**注:cmd_mask 裁剪规则部分需与 D6 同 batch 评审(co-seal),整个 E1 推迟到 Wave 8 启动以避免 D6 → E1 强前置违反** |
| E2 | Controller 结构设计 | `E-controller/E2-controller-structural.md` | 级联拓扑、共享 vs leaf、库块清单 |
| E3 | Controller 各环算法设计 | `E-controller/E3-loops-algorithm.md` | 位置/速度/姿态/角速度环数学定义、前馈、抗 windup、滤波器 |
| E4 | Controller 多旋翼 leaf 设计 | `E-controller/E4-multicopter-leaf.md` | 混控矩阵、几何、电机推力曲线对接、增益结构;**参数名与 firmware `CONTROL_PARAM` 字段名兼容性显式核对** |
| E5 | Controller 性能预算设计 | `E-controller/E5-performance-budget.md` | 5 ms 周期内各环执行预算、热点禁用项(MATLAB Function 等)、查找表与定点策略 |

### F. INS 契约消费(模型仓侧)

| ID | 名称 | 产出 | 退出条件 |
|---|---|---|---|
| F1 | ins_stub 功能设计 | `F-ins-contract/F1-ins-stub-functional.md` | 理想版/带扰版的输出特性、参数化(噪声/偏差/延迟)范围 |
| F2 | ins_stub 结构设计 | `F-ins-contract/F2-ins-stub-structural.md` | 内部块结构、数据流、`Plant_States_Bus` → `INS_Out_Bus` 字段映射表 |
| F3 | INS_Out_Bus 消费规则 | `F-ins-contract/F3-consumption-rules.md` | FMS/Controller 对 `INS_Out_Bus` 字段的依赖矩阵、validity 处理规则 |

### G. Harness 设计

| ID | 名称 | 产出 | 退出条件 |
|---|---|---|---|
| G1 | MIL 顶层结构设计 | `G-harness/G1-mil-toplevel.md` | 顶层 .slx 的子系统拓扑、引用模型策略、变体接入点 |
| G2 | 多速率调度设计 | `G-harness/G2-rate-scheduling.md` | Solver 配置、Sample Time 颜色检查清单、调度图 |
| G3 | 日志与可观测性设计 | `G-harness/G3-logging.md` | logsout 最小信号集(至少包含架构 v1 §13 各模块的代表性信号 + INS validity + cmd_mask + mode/state)、命名约定、采样时戳规则、单位 metadata、可视化目录、manifest 更新规则 |
| G4 | Pilot_Cmd 注入设计 | `G-harness/G4-pilot-injection.md` | 脚本驱动的 Pilot_Cmd 时间序列格式、场景描述 DSL(若需要) |

### H. 验证设计

| ID | 名称 | 产出 | 退出条件 |
|---|---|---|---|
| H1 | 验证场景目录设计 | `H-verification/H1-scenario-catalog.md` | golden path + edge case 场景全集与优先级;**每个场景标注是否可在未来 SIH 中重跑(应对架构 v1 §17 risk 5 时序失真)** |
| H2 | 验证指标设计 | `H-verification/H2-metrics.md` | 量化指标:位置/姿态/速率误差阈值、mode 切换正确性判据;**每个 H1 场景给出具体数值阈值(允许 TBD 占位但需说明定值的依据来源,例如"0.2 m: 来源于多旋翼 hover 静差预算")** |
| H3 | 回归基线策略设计 | `H-verification/H3-regression-baseline.md` | 基线 .mat 存放、版本化、对比脚本 IO;**RNG seed 管理契约(种子来源、固化方式、批量回归时的种子矩阵)** |
| H4 | 故障注入目录设计 | `H-verification/H4-fault-catalog.md` | 故障注入分类(传感器、链路、电池、估计器、电机、ENV);每类故障的注入接口、参数化范围、对应触发的 H1 场景 |

### I. 工具/脚本设计

| ID | 名称 | 产出 | 退出条件 |
|---|---|---|---|
| I1 | 仓库初始化脚本设计 | `I-tooling/I1-init-script.md` | `FMT_Model_Init.m` 的输入/输出/失败语义、加载顺序、清理策略;**至少一个 worked example(典型调用 + 期望 base workspace 状态)** |
| I2 | Bus/enum 镜像脚本设计 | `I-tooling/I2-bus-enum-mirror.md` | 从 firmware `*_types.h` 抽取规则、生成模板、人工 review 点 |
| I3 | 契约 diff 脚本设计 | `I-tooling/I3-contract-diff.md` | 与 B4 协同(co-seal):输入/输出格式、字段级 diff 报告、CI 接入方式;**实现 B4 全部覆盖项(bus/enum/PARAM/EXPORT/period/model_info + 符号存在性);至少一个 worked example(模拟一个差异输入,展示报告输出)** |
| I4 | Codegen 配置脚本设计 | `I-tooling/I4-codegen-config.md` | ert.tlc 完整配置项清单、per-module 差异、参数 inline 策略;**显式禁用项:动态内存、递归、可变尺寸信号(对应架构 v1 §4.5);代码格式与 lint 规则;至少一个 worked example** |
| I5 | 仿真运行/批量回归脚本设计 | `I-tooling/I5-sim-runner.md` | 单跑 / 批跑 / 回归三种模式的 CLI、报告输出;**记录架构 v1 §17 OQ5(导出自动化等级)的最终选择(手动 / 半自动 / 全自动);至少一个 worked example(每种模式的输入命令 + 期望产物)** |

## 5. 工作项数量与范围

- 总工作项数:**37** 个(A:**8**,B:5,C:4,D:6,E:5,F:3,G:4,H:**4**,I:5)
- 修订自 33 项 → 37 项 的依据:2026-05-05 平行审计报告 [_audit/2026-05-05-plan-audit.md](_audit/2026-05-05-plan-audit.md) 的 cross-cutting 缺口闭合(A6 init/reset、A7 time/clock、A8 共享库块、H4 故障注入)
- 不在本计划范围(明确推迟到实现阶段或后续阶段):
  - 任何 Simulink 模型实现
  - 任何脚本实现
  - codegen 实跑与导出
  - SIL/SIH/HIL 设计(除契约消费侧)
  - 多旋翼以外的机型 leaf 设计

## 6. 设计阶段的产出审阅闸口

每个工作项完成后需通过以下检查才能勾选退出:

1. **依赖一致性**:文档明确列出所引用的上游设计/契约,引用项已完成。
2. **契约影响标注**:若设计触及 firmware 可见接口,顶部加 `Contract impact: yes/no`,`yes` 则需在 [INDEX.md](INDEX.md) 决策日志登记。
3. **退出条件复核**:用本文件中该工作项的"退出条件"逐条对账。
4. **INDEX 更新**:在 `INDEX.md` 的"已完成设计文档清单"表追加一行。

## 7. 整体设计阶段退出条件

满足全部条件后,设计阶段才宣告结束,可启动实现阶段:

1. 37 个工作项全部完成且通过 §6 闸口。
2. [01-design-relationships.md](01-design-relationships.md) 中所有依赖边对应的两端文档都已完成。
3. 所有触及 firmware 契约的设计项在决策日志中有记录。
4. 用户对设计阶段进行一次整体复核并明确放行。

## 8. 设计阶段不做的事

- 不写 `.slx`、不写 `.m`(脚本设计文档可含伪代码,但不是可执行代码)。
- 不做工作量到天的精细排期(设计阶段长度由工作项推进决定,不预先承诺日期)。
- 不在设计文档中重复 firmware 已有的实现细节;只描述模型仓侧的设计。
- 不重复定义 bus/enum/参数 — 一律引用 B 区。
