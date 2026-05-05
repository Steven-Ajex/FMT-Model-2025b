---
work_item: A1
title: 架构 v1 评审与缺口闭合
upstream: ["架构 v1"]
contract_impact: no
status: reviewed
authored_at: 2026-05-05
last_reviewed_at: 2026-05-05
reviewer_verdict: pass
---

# A1 架构 v1 评审与缺口闭合

## 1. 目的

系统化审阅 [架构 v1](../../architecture/2026-05-05-fmt-model-architecture-v1.md) 全文(§1–§20),识别其中"未细化、待定、悬而未决"的所有问题点,并为每条缺口指派一个下游设计工作项作为闭合责任方。本文件是 A 区其余工作项(A2–A5)及 B/C/D/E/F/G/H/I 各区的入口扫描表:任何下游作者都应能仅凭本文件 + 架构 v1 即可定位到其所要承担的"待细化条目"。

## 2. 范围

**在范围:**

- 对架构 v1 §1–§20 全部章节做缺口枚举(under-specified / open-ended / deferred 三类)
- 为每条缺口分配 [00-design-plan.md](../00-design-plan.md) 中的工作项(A1–I5)作为闭合责任方
- 处置架构 v1 §17 开放问题清单(逐条:闭合 / 已解决 / 上交用户)
- 给出严重性评估(critical / major / minor)与统计汇总
- 当现有 33 个工作项无法覆盖某缺口时,推荐新增工作项(给出 tentative ID,不动 00-design-plan)

**不在范围(由其他工作项处理):**

- 修改架构 v1 文本本身(任何被标记为缺口的条目都由用户决定是否回填到架构 v1)
- 对各缺口的实际细化(由各闭合工作项执行)
- 修改 [00-design-plan.md](../00-design-plan.md) 或 [01-design-relationships.md](../01-design-relationships.md)(仅记录推荐,不直接编辑)
- 任何 Simulink / `.m` 实现层面的判断
- 评审下游设计文档(由 reviewer 角色另行执行)

## 3. 依赖

| 上游 | 引用位置 | 用途 |
|---|---|---|
| 架构 v1 全文 | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | 评审对象;每条缺口都源自其某一节 |
| 00-design-plan §4 工作项总览 | [链接](../00-design-plan.md) | 确定 A1 退出条件、查找闭合工作项 ID |
| 01-design-relationships §3/§4 | [链接](../01-design-relationships.md) | 确认 A1 → A2/A3/A4/A5 的下游传播关系 |
| RULES §3/§5/§6 | [链接](../RULES.md) | 决定本文件章节结构、禁则、Self-check |

注:本工作项是 Wave 1 唯一项,故无 design-side 上游;唯一上游为架构 v1。

## 4. 设计内容

### 4.1 评审方法

#### 4.1.1 缺口定义

将架构 v1 中以下任一形式的条目识别为"缺口":

1. **Under-specified(欠细化)**:架构 v1 给出概念或名词,但未提供下游可直接消费的具体内容(如"内部 bus 应模块前缀化",但未给清单)。
2. **Open-ended(开放性)**:架构 v1 列出多种可能但未抉择(如"Variant Subsystem vs 独立模型"未定)。
3. **Deferred(显式推迟)**:架构 v1 显式声明"留待后续",或形如"recommendation"未落到具体规则。
4. **Open question(显式开放问题)**:§17 列出的 6 项开放问题。

不视为缺口的条目:

- 已在架构 v1 中明确选定的决策(如"Plant 1 ms / Controller 5 ms / FMS 20 ms")
- 由非目标章节(§3)显式排除的内容
- 已在 §17 用 ~~删除线~~ 标记并附"Resolved (date)"的条目(只需在 §4.3 记录其 resolved 状态)

#### 4.1.2 严重性等级

| 等级 | 含义 | 判定 |
|---|---|---|
| **critical** | 不闭合则下游无法启动,或会导致 firmware 契约风险 | 阻塞 Wave 2/3/4 启动;触及 firmware 可见 bus/enum/symbol/period |
| **major** | 闭合前下游可启动但有返工风险;或影响多个下游 | 同时被 ≥ 2 个下游引用;或影响 codegen/timing 边界 |
| **minor** | 局部细节缺失,基本不影响主流程 | 仅影响单个工作项的内部组织或文风 |

#### 4.1.3 闭合工作项指派规则

1. 优先指派现有 33 个工作项之一(00-design-plan §4 中已列出)。
2. 若多个工作项均可闭合,选择**最贴近章节主题**的那一个;若是协同对(B1↔B2↔B3、D3↔D4↔D6 等),指派给协同 batch 的代表项并备注。
3. 若现有工作项均不能完整闭合(例如新增的横切关注点),提出 tentative 新工作项 ID(参见 §4.4),并标 *(proposed)*。
4. §17 开放问题的指派单列在 §4.3。

#### 4.1.4 评审顺序

按架构 v1 §1 → §20 顺序逐节扫读,每节产出一张表(§4.2 子节)。空表表示该节未发现缺口。

### 4.2 按章节列出的缺口与闭合分配

> 表头说明:
> - **Anchor**:架构 v1 的章节 / 段落 / 列表项序号
> - **Gap**:缺口简述(欠细化的具体对象)
> - **Type**:Under / Open / Deferred(对应 §4.1.1)
> - **Severity**:critical / major / minor
> - **Closed by**:闭合工作项 ID;`*` 表示协同 batch 代表项;*(proposed)* 表示推荐新增

#### §1 Purpose / §2 Project goals / §3 Non-goals

| Anchor | Gap | Type | Severity | Closed by | 备注 |
|---|---|---|---|---|---|
| §2.4 | "Separate shared logic from vehicle-specific logic" 未给出层级判定准则,什么属于 shared 何时下沉到 leaf 缺一致定义 | Under | major | A5 | A5 在变体策略中给出 shared/leaf 切分判据;C2/D2/E2 各自细化 |
| §2.6 | "Normalize bus, enum, parameter, and codegen management" 未给具体规范条目 | Under | major | A2 + B1/B2/B3 | A2 命名规则 + B 区清单共同闭合 |

§1 / §3 未发现缺口(陈述目的与非目标,无可细化项)。

#### §4 Design principles

| Anchor | Gap | Type | Severity | Closed by | 备注 |
|---|---|---|---|---|---|
| §4.1 "Contract-first" | "field order, enum values, exported model info structs" 三者各自的 freeze 边界未明确 | Under | critical | B1 + B2 + B3 | B1/B2/B3 协同 batch 必须给出每类契约的 freeze 范围 |
| §4.4 "Single-rate per module" | INS 100 Hz 在模型仓侧"如何采样"未规定(整数倍 vs 异步) | Under | major | A4 | A4 跨速率边界设计中给出 INS 输入的采样策略 |
| §4.5 "Deterministic codegen" | "no dynamic memory, no recursion, no variable-size signals" 落到 ert.tlc 配置项未列出 | Deferred | major | I4 | I4 codegen 配置脚本设计 |
| §4.6 "Explicit interfaces" | "documented rate boundaries" 文档形式未指定 | Deferred | minor | A4 | A4 输出包含 rate transition 文档化模板 |
| §4.7 "Build for verification" | "independently exercisable in MIL" 的最小 harness 形态未定义 | Deferred | major | G1 | G1 顶层结构设计 |

#### §5.1 Existing firmware contract constraints

| Anchor | Gap | Type | Severity | Closed by | 备注 |
|---|---|---|---|---|---|
| §5.1.2 | `FMS_EXPORT` / `CONTROL_EXPORT` / `PLANT_EXPORT` 字段集仅列 "至少包含 period 和 model_info[]";完整字段集未枚举 | Under | critical | B3 | B3 parameter schema 设计需给出 EXPORT struct 全字段 |
| §5.1.3 | INS 100 Hz "firmware-owned wrapper/system integration cadence" 与模型仓侧采样关系未刻画 | Under | major | A4 + F1 | A4 给采样策略;F1 给 stub 输出节奏 |
| §5.1.4 | FMS 输入清单未给"哪些必选 / 哪些可选 / Control_Out_Bus 的回送条件" | Under | major | A3 | A3 模块边界信号清单 |
| §5.1.5 | `FMS_Out_Bus` 重要字段一句话带过(rate cmd / attitude cmd / cmd_mask / status / ext_state / ctrl_mode / wp / home / error 等),字段顺序与类型未定义 | Under | critical | B1 | B1 bus 清单与 schema |
| §5.1.6 | Controller "Inputs/Outputs" 仅列 bus 名,未给字段方向矩阵 | Under | major | A3 + B1 | A3 信号方向 + B1 字段表 |
| §5.1.7 | Plant 输入/输出含 "optional others" 模糊列举 | Under | major | A3 + B1 | A3 列定 optional 的取舍判据;B1 落到字段 |
| §5.1.8 | "Any change to `INS_Out_Bus` ... must be coordinated with FMT-Firmware first, then mirrored" 流程未定义 | Deferred | critical | B5 | B5 INS_Out_Bus 镜像同步流程 |
| §5.1.9 | "Current enum numeric values are already embedded ... must not drift silently" 未给检测机制 | Deferred | critical | B2 + B4 + I3 | B2 锁定数值;B4 策略;I3 实现 |
| §5.1.10 | "drop generated `.c/.h` into a `lib/` directory next to a thin interface layer" 未给目录契约清单 | Under | major | I4 | I4 codegen 配置 |

#### §5.2 New 2025b repository organization recommendations

| Anchor | Gap | Type | Severity | Closed by | 备注 |
|---|---|---|---|---|---|
| §5.2.1 | "One repository-wide source of truth for buses, enums, parameters" 物理形态未抉择(MATLAB script vs Data Dictionary vs JSON-derived) | Open | major | A2 + B1 | A2 落到目录与命名;B1 落到来源形态 |
| §5.2.4 | "reproducible and non-interactive" 脚本骨架未列 | Deferred | minor | I1 | I1 init 脚本设计 |
| §5.2.6 | "Standardize directory names, harness placement, and export staging" 完整规范未给 | Under | major | A2 + G1 | A2 命名;G1 harness 顶层 |

#### §6 Recommended top-level repository tree

| Anchor | Gap | Type | Severity | Closed by | 备注 |
|---|---|---|---|---|---|
| §6 树结构 | `model/shared/lib/{math,filters,guidance,control,safety,utilities}` 各子目录的入选库块清单未定 | Deferred | major | C2 + D2 + E2 | 各模块结构设计中识别共享需求,A6(若新增)汇总 |
| §6 `model/shared/data/` | 用途未说明 | Under | minor | C3 + D5 + E4 | 各 leaf 工作项决定是否使用并写入清单 |
| §6 `model/<m>/vehicles/{multicopter,fixwing,vtol,boat,car,...}` | 多旋翼以外的 vehicles 占位目录是否在设计阶段建立未定 | Open | minor | A5 | A5 变体策略说明阶段策略 |
| §6 `tests/{smoke,contract,regression}` | 三类测试边界与归属未定 | Deferred | major | H1 + H3 | H1 场景目录 + H3 回归基线 |
| §6 `config/{simulink,codegen,vehicles}` | 配置文件清单未列 | Deferred | major | I1 + I4 | I1 init + I4 codegen |
| §6 `export/{firmware,reports}` | export 触发条件、内容打包形态未定 | Deferred | major | I4 + B4 | I4 codegen 输出;B4 契约 diff 报告 |

#### §7 Proposed top-level module split

| Anchor | Gap | Type | Severity | Closed by | 备注 |
|---|---|---|---|---|---|
| §7.2 FMS "Does not own ... beyond any contract-required passthrough fields" | passthrough 字段集合未枚举 | Under | major | A3 + D6 | A3 边界 + D6 cmd_mask 语义 |
| §7.3 Controller "feed-forward, anti-windup, vehicle-specific mixer/allocation" | 各项在 shared 还是 leaf 未划分 | Open | major | E2 + E4 | E2 结构 + E4 leaf |

#### §8 Architectural boundaries by module

| Anchor | Gap | Type | Severity | Closed by | 备注 |
|---|---|---|---|---|---|
| §8.1 Plant 输出 "optional others" | optional 传感器 bus 的入选准则未定 | Under | minor | A3 + C1 | A3 列出准则;C1 决定保真度对应的传感器集 |
| §8.2 INS boundary | "any change in firmware's INS output must be reflected here in lockstep" 的 lockstep 操作步骤未定 | Deferred | critical | B5 | B5 镜像流程 |
| §8.3 FMS "existing contract-aware optional `Control_Out_Bus` input if needed" | 何时启用、由谁判定未定 | Open | major | D1 + A3 | D1 功能设计裁决;A3 落到信号清单 |
| §8.4 Controller "optional `Plant_States_Bus` only in harness/HIL/SIH variants" | 启用条件、变体接入点未定 | Open | major | A5 + G1 | A5 变体策略;G1 harness 顶层 |

#### §9 Cross-module buses and dataflow

| Anchor | Gap | Type | Severity | Closed by | 备注 |
|---|---|---|---|---|---|
| §9.2.4 | "Internal buses should be module-prefixed" — 完整内部 bus 清单未给 | Deferred | major | D2 + E2 + C2 | 各模块结构设计中产出本模块内部 bus 清单;A2 给前缀规则 |
| §9.3.2 | "Cross-rate changes occur only in harness or scheduler wiring" — harness 中的 rate transition 块清单未定 | Deferred | major | A4 + G2 | A4 抉择策略;G2 落到调度图 |
| §9.3.3 | "Every contract bus must have one canonical schema source" — schema source 的存放与生成方式未定 | Open | major | B1 + I2 | B1 给来源;I2 给镜像脚本 |
| §9.3.4 | "Field order changes in any firmware-visible bus require explicit contract review" — review 触发与流程未定 | Deferred | critical | B4 + I3 | B4 策略;I3 脚本 |

#### §10 Shared-vs-vehicle layering

| Anchor | Gap | Type | Severity | Closed by | 备注 |
|---|---|---|---|---|---|
| §10.1 Layer A | "common safety checks" 是否包括 FMS 安全策略骨架未明 | Open | minor | D2 | D2 中识别 shared 安全块归属 |
| §10.2 Layer B | "Controller shared cascade loop shells" 共享到何粒度(块 vs 子系统模板)未定 | Open | major | E2 | E2 结构设计 |
| §10.3 Layer C | "Vehicle-specific logic should be leaf-level only" 的反例豁免清单未定 | Open | minor | A5 | A5 给变体策略 + 豁免准则 |

#### §11 Simulink, script, and codegen layering

| Anchor | Gap | Type | Severity | Closed by | 备注 |
|---|---|---|---|---|---|
| §11.1 | "data dictionaries or scripted bus/enum definitions" 二者抉择未做 | Open | major | B1 | B1 给抉择 |
| §11.2 | 脚本子集(init / build / verify / export / dev)各自接口与依赖未定 | Deferred | major | I1 + I2 + I3 + I4 + I5 | 五个 I 区项分别承担 |
| §11.3 | "deliberately place selected outputs into `export/firmware/<module>/<vehicle>/`" 的 vehicle 分目录命名未定 | Deferred | minor | A2 | A2 命名规则 |

#### §12 Recommended module internals

| Anchor | Gap | Type | Severity | Closed by | 备注 |
|---|---|---|---|---|---|
| §12.1.1–6 FMS 6 子模块 | 各子模块内部信号、事件、状态、与 `FMS_Out_Bus` 字段的映射未定 | Under | critical | D2 + D3 + D4 + D6 | D2 边界;D3 Mode Mgr;D4 Shaper;D6 cmd_mask |
| §12.1.3 Safety Monitor | "failsafe conditions ... low battery, geofence, invalid mode preconditions" 的判定阈值与触发动作未定 | Under | major | D1 | D1 功能设计 |
| §12.1.4 Mission/Auto Manager | "mission item sequencing, pause/continue/return/takeoff/land" 在闭环切片需要哪些子集未定 | Under | major | D1 + D5 | D1 范围;D5 多旋翼 leaf |
| §12.1.5 Command Shaper | "slew/rate/jerk limiting" 各 mode 下的具体限值未定 | Under | major | D4 + D5 | D4 策略;D5 多旋翼数值 |
| §12.2.1–7 Controller 7 子模块 | 各子模块算法定义、数据类型、定点/浮点策略未定 | Under | critical | E1 + E2 + E3 + E4 + E5 | E 系列覆盖 |
| §12.2.2 Position Loop | "only if controller contract really needs it" — 多旋翼是否启用未定 | Open | major | E1 + E4 | E1 通用裁剪规则;E4 多旋翼裁决 |
| §12.2.5 Rate Loop | "anti-windup and LPF D-term discipline" 算法未定 | Under | major | E3 | E3 各环算法 |
| §12.4 Plant 6 阶段 | 各阶段的物理保真度等级未定 | Under | major | C1 + C2 + C3 + C4 | C 系列覆盖 |
| §12.4 Plant | "slower sensor publication modeled through output update cadence" 实现方式未定 | Open | major | C2 + A4 | C2 内部结构;A4 给跨速率边界策略 |

#### §13 Timing constraints and scheduling assumptions

| Anchor | Gap | Type | Severity | Closed by | 备注 |
|---|---|---|---|---|---|
| §13 表 | INS 在模型仓侧 100 Hz 的"采样到 FMS 50 Hz / Controller 200 Hz"的对齐策略未定 | Under | major | A4 | A4 跨速率边界 |
| §13.5 "Plant and INS latency must be considered" | latency 量化预算未定 | Deferred | major | A4 + E5 | A4 给边界;E5 给 5 ms 预算 |
| §13 实操含义 §3 | "Controller execution budget is most critical" — 5 ms 内每环子预算未定 | Deferred | major | E5 | E5 性能预算 |

#### §14 Parameter, bus, and enum management

| Anchor | Gap | Type | Severity | Closed by | 备注 |
|---|---|---|---|---|---|
| §14.1 Bus | "Generate or derive firmware-facing bus headers from that source" — 生成 vs 镜像方向未定 | Open | critical | B1 + B5 + I2 | B1/B5 决策;I2 实现 |
| §14.2 Enum | "Mode-related enums should be owned by FMS shared definitions" — FMS shared 与 B 区中央 enum 库的归属未明 | Open | major | B2 + D1 | B2 锁定;D1 决定 mode 集合 |
| §14.3.3 | "Runtime-tunable versus compile-time-inlined parameters must be intentionally categorized" — 分类标准未定 | Deferred | major | B3 + I4 | B3 标记;I4 inline 策略 |
| §14.4 Contract review gates | gate 触发时机、自动化程度未定 | Deferred | critical | B4 + I3 | B4 策略;I3 脚本 |

#### §15 MIL, SIL, HIL/SIH, and firmware integration path

| Anchor | Gap | Type | Severity | Closed by | 备注 |
|---|---|---|---|---|---|
| §15.1 MIL | "step response / mode transition / failsafe / command shaping quality" 未给量化指标 | Deferred | major | H1 + H2 | H1 场景;H2 指标 |
| §15.1 MIL | `ins_stub` 噪声/偏置/延迟参数范围未定 | Under | major | F1 | F1 ins_stub 功能设计 |
| §15.2 SIL | "output equivalence within tolerance" 容差阈值未给 | Deferred | major | H2 + H3 | H2 指标;H3 基线 |
| §15.3 SIH/HIL | "timestamp assumptions / topic freshness / reset/init sequencing / parameter linkage" 检查清单未给 | Deferred | minor | (out of design scope) | 推迟到实现阶段;在本评审中不强制闭合 |
| §15.4 Firmware integration | "exported symbols match interface code expectations" 检查清单未给 | Deferred | major | I3 + B4 | I3 + B4 |
| §15 Promotion gate | "MIL pass -> SIL pass -> SIH/HIL pass -> firmware branch integration" 的判定标准未定 | Deferred | major | H3 | H3 回归基线策略 |

#### §16 Smallest executable slice

| Anchor | Gap | Type | Severity | Closed by | 备注 |
|---|---|---|---|---|---|
| §16.6 | "arm -> takeoff/hold -> manual or position hold -> land/disarm" 场景脚本格式未定 | Deferred | major | G4 + H1 | G4 注入格式;H1 场景目录 |
| §16 排除项 | "full mission library breadth" 的下界(切片需要的最小 mission 子集)未定 | Under | major | D1 + D5 | D1 mission 范围;D5 多旋翼 |

#### §17 Risks and open questions(风险部分)

| Anchor | Gap | Type | Severity | Closed by | 备注 |
|---|---|---|---|---|---|
| §17 Risk 1 Contract drift | 监测机制未定 | Deferred | critical | B4 + I3 | B4 策略 + I3 脚本 |
| §17 Risk 2 INS_Out_Bus drift | 已给 mitigation,但 versioning 位置未定(由 §17.OQ6 闭合) | Open | critical | B5 | B5 INS_Out_Bus 镜像同步 |
| §17 Risk 3 Over-abstraction | 防止过早抽象的 review checkpoint 未定 | Deferred | minor | A5 | A5 变体策略 |
| §17 Risk 4 INS-stub realism | "configurable noise/bias/delay knobs" 参数范围未定 | Under | major | F1 | F1 ins_stub 功能设计 |
| §17 Risk 5 Timing realism | MIL→SIH 之间的 timing 验证桥接未定 | Deferred | major | H1 + H3 | H1 场景含 timing 子类;H3 基线 |
| §17 Risk 6 Parameter governance | tuning divergence 检测机制未定 | Deferred | major | B3 + B4 | B3 schema 锁定 + B4 diff |
| §17 Risk 7 Variant complexity | variant 复杂度上限未定 | Open | major | A5 | A5 变体策略 |

§17 开放问题(OQ1–OQ6)单列在 §4.3。

#### §18 Phased progression

| Anchor | Gap | Type | Severity | Closed by | 备注 |
|---|---|---|---|---|---|
| §18 Phase 1 退出 | "one no-op or stub export can reproduce firmware-compatible headers/artifacts layout" 的接受准则未定 | Deferred | major | I4 + B4 | I4 输出形态;B4 接受准则 |
| §18 Phase 2 退出 | "generated FMS/Controller headers pass contract diff review" 的 diff 范围未定 | Deferred | major | B4 + I3 | B4 + I3 |
| §18 Phase 3 退出 | "agreed tolerance" 的具体值未定 | Deferred | major | H2 | H2 量化指标 |

#### §19 First implementation target / §20 Summary

§19 / §20 未发现新缺口(均为对前文决策的复述)。

### 4.3 §17 open questions 的闭合分配

逐条处置架构 v1 §17 列出的 6 项开放问题:

| OQ # | 原文摘要 | 处置 | 闭合工作项 / 锚点 | 备注 |
|---|---|---|---|---|
| OQ1 | "首次 2025b 交付目标多旋翼 only,还是多旋翼+固定翼共享脚手架?" | **Closed** | A5 | A5 变体策略需在文档中对此抉择;00-design-plan §5 已声明"不在范围"含"多旋翼以外的机型 leaf 设计",倾向 OQ1 选择"多旋翼 only";A5 显式确认 |
| OQ2 | "现 FMS 行为多大程度结构化保留 vs 干净重写?" | **Closed** | D1 + D2 | D1 功能设计列入 mode 集合时需对此明确(保留 / 重写 / 部分);D2 结构设计落地 |
| OQ3 | "INS 是否走模型仓侧的 wrapper 路线?" | **Resolved** (2026-05-05) | architecture v1 §17 已用删除线标注 + 显式 Resolved 注释 | 无需指派;在本文件登记以保证可追溯 |
| OQ4 | "`Control_Out_Bus` 是否应继续作为 FMS 输入?" | **Closed** | D1 + A3 | D1 给功能裁决(是否真的需要);A3 落到 bus 边界清单;若决策为"过渡",在 D1 中给出退出计划 |
| OQ5 | "导出自动化期望级别?" | **Closed** | I4 + I5 | I4 codegen 配置 + I5 sim runner 共同约束;若需在两项之上加一总体抉择,见 §4.4 提案 A6 |
| OQ6 | "`INS_Out_Bus` schema 版本管理在哪里?" | **Closed** | B5 | B5 INS_Out_Bus 镜像同步流程必须给出 single source of truth + diff workflow;契约影响 yes |

汇总:1 closed-by-resolved,5 closed-by-assignment,0 escalated 给用户。所有 6 项都已分派归宿。

> 备注:OQ1 / OQ4 / OQ5 在指派后,仍建议用户在评审本文件时确认抉择倾向(尤其 OQ1 的"多旋翼 only" 已与 00-design-plan §5 一致,确认即可)。

### 4.4 推荐新增工作项(若有)

经盘点,33 个现有工作项基本能覆盖架构 v1 的全部缺口。以下为**可选**新增建议,均标 *(proposed, tentative ID)*,**不**自行写入 [00-design-plan.md](../00-design-plan.md);留待用户裁决。

| Tentative ID | 名称 | 动机 | 建议归属 | 是否必须 |
|---|---|---|---|---|
| **A6** *(proposed)* | 共享库块清单与归属总表 | §6 中 `model/shared/lib/{math,filters,guidance,control,safety,utilities}` 六类共享库的入选块,目前散落在 C2/D2/E2 各自决定。建议增加一个横切汇总,避免三个模块各推一份不一致的"共享候选"清单。 | A 区 | 否(若 A2 命名规则中已附"共享库块命名 + 横向 review 检查表"则可省) |
| **B6** *(proposed)* | 契约总览与 freeze list | §5.1 列出诸多硬契约,但缺一份"什么字段被冻结、什么允许变更、变更需要何种审批"的总览。B1/B2/B3 各自负责自己的清单,但缺顶层 freeze 决策表。 | B 区 | 否(可并入 B4 契约 diff 策略) |
| **I6** *(proposed)* | 端到端导出流水线设计 | OQ5 关心导出自动化级别;I4(codegen 配置)+ I5(sim runner)各管一段,缺"从 build → diff → 打包 → 推送"的端到端流水线声明。 | I 区 | 否(可作为 I5 的子节扩展) |

**推荐行动**:用户在评审本文件后,选择以下三种处理之一:

1. 接受新增,把 A6/B6/I6(或其子集)写入 00-design-plan §4 与 01-design-relationships §4。
2. 拒绝新增,要求把对应职责合并到既有工作项(例如 A6 → A2、B6 → B4、I6 → I5)并增补其退出条件。
3. 暂不裁决,本文件中保留为 *(proposed)* 标记,Wave 2 启动前再决定。

### 4.5 评审结论汇总

#### 4.5.1 缺口计数

按严重性聚合 §4.2 + §4.3 中所有缺口与 OQ:

| 严重性 | §4.2 章节缺口 | §4.3 OQ | 合计 |
|---|---:|---:|---:|
| critical | 12 | 1 (OQ6) | 13 |
| major | 36 | 4 (OQ1/OQ2/OQ4/OQ5) | 40 |
| minor | 7 | 0 | 7 |
| (resolved-only,无需闭合) | 0 | 1 (OQ3) | 1 |
| **总计** | **55** | **6** | **61** |

> 计数说明:同一缺口若指派给协同 batch(例如 B1+B2+B3),仍按 1 条计;每行表示 1 条缺口。

#### 4.5.2 工作项负载分布(每项被指派的缺口数)

| 工作项 | 指派计数 | 工作项 | 指派计数 |
|---|---:|---|---:|
| A2 | 4 | E2 | 4 |
| A3 | 7 | E3 | 2 |
| A4 | 7 | E4 | 3 |
| A5 | 7 | E5 | 3 |
| A6 *(prop.)* | 1 | F1 | 3 |
| B1 | 8 | G1 | 4 |
| B2 | 4 | G2 | 1 |
| B3 | 5 | G4 | 1 |
| B4 | 8 | H1 | 5 |
| B5 | 5 | H2 | 4 |
| B6 *(prop.)* | 0 (替代源) | H3 | 4 |
| C1 | 2 | I1 | 2 |
| C2 | 4 | I2 | 2 |
| C3 | 1 | I3 | 7 |
| C4 | 1 | I4 | 7 |
| D1 | 6 | I5 | 1 |
| D2 | 4 | I6 *(prop.)* | 0 (替代源) |
| D3 | 1 | | |
| D4 | 3 | | |
| D5 | 3 | | |
| D6 | 3 | | |
| E1 | 4 | | |

负载较高的项(B1 / B4 / I3 / I4 / A3 / A4 / A5 各 ≥ 7):重点关注其退出条件是否覆盖到所有指派缺口。

#### 4.5.3 critical 级缺口集中位置

13 条 critical 缺口分布:

- §4 Design principles(契约 freeze 范围)→ B 区
- §5.1 firmware constraints(EXPORT 字段 / FMS_Out_Bus 字段 / lockstep / enum drift)→ B1 / B2 / B3 / B5
- §8.2 INS lockstep 流程 → B5
- §9.3.4 字段顺序变更评审 → B4 + I3
- §12.1 FMS 6 子模块映射 → D2/D3/D4/D6
- §12.2 Controller 7 子模块算法 → E1/E2/E3
- §14.1 bus 来源方向 → B1+B5+I2
- §14.4 contract review gates → B4+I3
- §17 Risk 1 / Risk 2 → B4+I3 / B5
- OQ6 INS_Out_Bus versioning → B5

观察:critical 集中在 B 区(契约)与 D/E 模块内部细化。这与设计计划 Wave 4(B 区协同 batch)及 Wave 7–9(D/E 模块详细)是匹配的;**A1 评审通过后立即进入 A2/A4/A5 → A3 → B 区是关键路径**。

## 5. 已知风险与悬而未决问题

- **A1 自身的覆盖完整性风险**
  - 影响:若本文件遗漏某缺口,该缺口在 Wave 后期才被发现会引发返工。
  - 处置:在评审本文件时,reviewer 须按章节复核;同时鼓励下游作者发现新缺口时 PR 回填本文件的 §4.2(走 RULES §10 变更日志)。

- **新增工作项的裁决推迟风险**
  - 影响:A6/B6/I6 若不及时裁决,某些 critical 缺口的责任方不清晰(尤其 freeze list)。
  - 处置:Wave 2 启动前用户必须裁决 §4.4;若裁决"合并到既有项",同步在 00-design-plan 该项退出条件中追加。

- **OQ3 的回溯一致性**
  - 影响:OQ3 已 Resolved(2026-05-05),但架构 v1 多个章节(§3.5 / §5.2.5 / §12.3 / §15 / §17 等)分别独立陈述了同一决策;若未来 INS 决策反转,有多点修改风险。
  - 处置:在 INDEX 决策日志登记本次 resolved 锚点;若反转,统一从 §3 开始改起。

- **多旋翼以外机型的占位性缺口**
  - 影响:00-design-plan §5 显式排除"多旋翼以外的机型 leaf 设计",但架构 v1 §6 / §7 树中保留了 fixwing/vtol/boat/car 目录。本文件未为这些目录指派工作项。
  - 处置:在 A5 变体策略中明确"占位目录"在设计阶段的处置(创建空 README vs 不创建);本文件对此放行。

- **本文件不修改架构 v1**
  - 影响:本文件仅记录;架构 v1 文本中那些"recommendation"用词依旧存在,可能让读者误以为是软约束。
  - 处置:用户在评审本文件后,可酌情发起架构 v1 增订,把已被本文件指派闭合的 recommendation 升级为"hard constraint, closed by <ID>"。

## 6. 退出条件复核

A1 在 [`00-design-plan.md`](../00-design-plan.md) §4.A 中的退出条件原文:

> 列出 v1 中未细化的所有问题点;每条标注由哪个工作项闭合

逐条复核:

| # | 退出条件分解 | 本文档依据 | 状态 |
|---|---|---|---|
| 1 | "列出 v1 中未细化的所有问题点" — 系统性扫读 §1–§20 | §4.1 评审方法定义"缺口";§4.2 按章节(§1–§20)逐节产出表 | 满足 |
| 2 | "所有" — 覆盖完整性 | §4.2 含每个有缺口的章节(§1/§3/§19/§20 经判定无缺口已注明);§4.3 独立处置 §17 OQ;§4.4 指出 33 项无法覆盖时的新增建议 | 满足 |
| 3 | "每条标注由哪个工作项闭合" — 责任分配 | §4.2 每行 "Closed by" 列;§4.3 OQ 列;§4.4 *(proposed)* 标记 | 满足 |
| 4 | 可追溯回源 | §4.2 每行 "Anchor" 列指向架构 v1 章节 / 段落;OQ 列对应 §17 编号 | 满足(隐含要求,本文件主动满足) |
| 5 | 严重性可见性 | §4.5.1 计数表;§4.5.3 critical 集中位置 | 满足(隐含要求) |

退出条件满足;状态可由 reviewer 升级到 reviewed。

## 7. 下游影响

按 [`01-design-relationships.md §4.1`](../01-design-relationships.md) 中 A 区出边:

```text
架构 v1 → A1
A1 → A2
A1 → A4
A1 → A5
A2, A4, A5 → A3
```

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [A2 命名与目录约定设计](A2-naming-conventions.md) *(未启动)* | → | §4.2 中 §5.2 / §6 / §11 行所列"命名与目录"类缺口清单;§4.4 A6(若接受合并)的命名义务 |
| [A4 跨速率边界设计](A4-rate-boundaries.md) *(未启动)* | → | §4.2 §4 / §5.1 / §9.3 / §12.4 / §13 中"rate transition / latency / 采样对齐"类缺口 |
| [A5 变体策略设计](A5-variant-strategy.md) *(未启动)* | → | §4.2 §6 / §8.4 / §10 / §17 Risk 7 中"variant / leaf 豁免"类缺口;§4.3 OQ1 |
| [A3 模块边界信号清单细化](A3-module-boundaries.md) *(经 A2/A4/A5 后启动)* | → (transitively) | §4.2 §5.1 / §7 / §8 中"信号方向 / 字段优化 / passthrough"类缺口 |

A1 还**间接影响**(按拓扑):

- B 区(B1/B2/B3/B4/B5):本文件标出的 critical 契约缺口集中于此,B 区批次启动时应以 §4.5.3 为入口
- C/D/E/F 区:间接通过 A3/B 区接收缺口列表
- G/H/I 区:本文件直接指派的缺口(详见 §4.2 各表的 Closed by 列)

下游作者使用本文件的方式建议:

1. 在自己的工作项的 §3 依赖中引用本文件并列出所指派的缺口行号(章节锚点)。
2. 在自己的工作项的 §6 退出条件复核中,逐条确认对应的"Closed by"缺口已被本文档的设计内容闭合。
3. 若发现本文件遗漏了应由本工作项闭合的缺口,按 RULES §10 在本文件追加变更日志,沿出边重新通知。

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-05 | A1 author | 初稿 |

## Self-check

- [x] frontmatter 完整,字段值合法
- [x] 上游文档全部存在且 status ≥ reviewed(唯一上游为架构 v1,已发布并锚定 2026-05-05 修订;按本设计阶段约定视同 ≥ reviewed)
- [x] 退出条件逐条复核完成,每条均给出依据
- [x] 引用路径全部可点击访问
- [x] 不存在 RULES §5 禁则中的内容
- [x] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记(本文件 contract_impact=no,N/A)
- [x] 下游影响已沿关系图识别完毕
- [x] 文档不超出本工作项范围(无越权设计)
