---
work_item: A5
title: 变体策略设计
upstream: ["架构 v1", "A1"]
contract_impact: no
status: reviewed
authored_at: 2026-05-05
last_reviewed_at: 2026-05-05
reviewer_verdict: pass
---

# A5 变体策略设计

## 1. 目的

为 FMT-Model-2025b 三个模型仓侧顶层模块(Plant / FMS / Controller)与 Phase 2 多旋翼切片确立一套**变体管理策略框架**:决定每个顶层模型采用哪种 Simulink 复用机制(Model Reference / Library / Subsystem),冻结 Phase 2 唯一允许的变体轴并显式排除其他维度,设定"过度抽象预算上限"防止 §17 Risk 3,并落地架构 v1 §17 OQ1(vehicle scope)的最终选择。本文件**只设计变体框架**,不设计任何具体 vehicle leaf 的内部内容(那是 C3/D5/E4)。

## 2. 范围

**在范围:**

- (a) Plant / FMS / Controller 三个顶层模型的 Model Reference vs Library vs Subsystem 选型与依据
- (b) Phase 2 唯一变体轴(vehicle class)的枚举与显式冻结其他变体轴(sensor / mission / actuator count)
- (c) 过度抽象预算上限(单 Variant 块支持 vehicle 数 / leaf 文件数的硬上限),含 §17 Risk 3 review checkpoint
- (d) 架构 v1 §17 OQ1(multicopter only vs +fixed-wing scaffolding)的最终决策与依据
- 多旋翼以外 vehicle 占位目录在设计阶段的处置(§6 树中 fixwing/vtol/boat/car 目录)
- 与 §17 Risk 7(Variant complexity)对应的复杂度治理规则
- shared/leaf 切分判据(闭合 A1 §4.2 §2.4 / §10.3 缺口)
- Controller 的 optional `Plant_States_Bus` 变体接入点框架(闭合 A1 §4.2 §8.4 缺口,具体 harness 落点交 G1)

**不在范围(由其他工作项处理):**

- 多旋翼具体几何 / 质量惯量 / 分配矩阵参数 — 由 [C3 Plant 多旋翼 leaf 设计](C3-multicopter-leaf.md) 处理
- 多旋翼具体 mode-cmd 映射 / 起飞降落 profile — 由 [D5 FMS 多旋翼 leaf 设计](D5-multicopter-leaf.md) 处理
- 多旋翼具体混控矩阵 / 增益 — 由 [E4 Controller 多旋翼 leaf 设计](E4-multicopter-leaf.md) 处理
- 各模块内部 shared 子系统层次结构 — 由 C2 / D2 / E2 处理
- harness 顶层装配中 variant 接入点的物理拓扑 — 由 [G1 MIL 顶层结构设计](../G-harness/G1-mil-toplevel.md) 处理
- bus / enum / param 字段表 — B 区已锁定,本文件只引用
- ert.tlc 配置项落地 — 由 I4 处理(本文件只声明对 codegen 的策略约束)

## 3. 依赖

| 上游 | 引用位置 | 用途 |
|---|---|---|
| 架构 v1 §6 仓库树 | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | `model/<m>/vehicles/{multicopter,fixwing,vtol,boat,car}` 占位目录是变体策略的直接对象 |
| 架构 v1 §10 shared-vs-vehicle | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | 三层(A 共享核 / B 模块共享架构 / C 机型 leaf)是 shared/leaf 切分判据基底 |
| 架构 v1 §16 smallest slice | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | "multicopter only" 切片是 (b) 单一变体轴与 (d) OQ1 决策的核心依据 |
| 架构 v1 §17 Risk 3 / Risk 7 | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | (c) 过度抽象预算与 (b) 变体冻结分别针对这两条风险 |
| 架构 v1 §17 OQ1 | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | (d) 直接闭合 OQ1 |
| 架构 v1 §18 Phase 2 / Phase 5 | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | Phase 2 仅多旋翼;Phase 5 才扩展;变体策略需为 Phase 5 留 forward path 但不实现 |
| 架构 v1 §19 first target | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | "multicopter-only vertical slice" 是 (d) 的官方答案出处 |
| [A1 架构 v1 评审与缺口闭合](A1-v1-review.md) | reviewed (2026-05-05) | "Closed by A5" 行(§4.2 §2.4 / §6 / §8.4 / §10.3 / §17 Risk 3 / §17 Risk 7 + §4.3 OQ1)是本文件必须闭合的具体缺口清单 |
| 00-design-plan §4.A | [链接](../00-design-plan.md) | A5 退出条件原文(含 2026-05-05 收紧后的 (a)/(b)/(c)/(d) 子项) |

## 4. 设计内容

### 4.1 (a) Plant / FMS / Controller 顶层选型:Model Reference vs Library vs Subsystem

#### 4.1.1 三种机制的对比维度

| 维度 | Model Reference | Library(Linked) | (Plain) Subsystem |
|---|---|---|---|
| 独立 codegen 单元 | 是(每个被引用模型一份独立生成) | 否(随宿主一起生成) | 否 |
| 独立 build 缓存 | 是(`slprj` 下分目录) | 部分 | 否 |
| 接口强约束(Bus / param) | 强(必须显式声明 Inport/Outport bus + Model Workspace) | 弱(凭名称解析) | 弱 |
| Cross-vehicle reuse | 高(可在多 harness / vehicle 间作为已编译单元复用) | 高(库块层面) | 低(必须复制粘贴) |
| Build time(增量) | 优(未变模型不重 build) | 中 | 差(顶层一动全 build) |
| Debugability(单步) | 中(进入 referenced model 需要切换) | 优(library 块可在 host 上下文调试) | 优(直接进入) |
| firmware drop-in 匹配度 | 高(每模块一份独立 `<Module>.c/.h`,正合 §5.1.10 lib/ 目录契约) | 低(库块嵌入时不形成 per-module 顶层) | 低 |
| 设置成本 | 高(Model Workspace、Configuration Reference、bus 编辑) | 低 | 极低 |

#### 4.1.2 决策

**三个顶层模型(Plant / FMS / Controller)**:**采用 Model Reference 作为顶层封装机制**(每个模型即对应 firmware 期望的一套 `<Module>_init`/`<Module>_step`/`<MODULE>_EXPORT`/`<MODULE>_PARAM` 产出)。

依据:

1. **firmware drop-in 模式匹配**:架构 v1 §5.1.2 / §5.1.10 要求每个模型生成 `<Module>.c/.h` 落入 firmware 对应模块的 `lib/` 目录,Model Reference 是唯一直接产出独立 codegen 单元的机制(Library 与 Subsystem 都会被宿主 inline)。
2. **独立 codegen 表面与契约 diff 匹配**:B4(契约 diff 策略)需要逐模块比对 EXPORT / PARAM / 符号存在性(`FMS_init`/`FMS_step` 等),Model Reference 的产物粒度天然对齐这一比对边界。
3. **增量 build / 加快开发循环**:Plant 改动不应重 build FMS / Controller;Model Reference 的 `slprj` 缓存提供这一隔离。
4. **Cross-vehicle reuse**:Phase 5 扩展第二机型时,Plant / FMS / Controller 顶层 referenced model 可在 vehicle harness 间复用,不需要复制顶层装配。
5. **接口强约束契合 contract-first 原则**:架构 v1 §4.1 contract-first;Model Reference 的 Inport/Outport 必须显式 bus 化,正好把契约风险挡在编辑期。

**模块内部 shared / vehicle leaf 子系统的机制**(本文件给框架,具体由 C2/D2/E2 落地):

- **`model/shared/lib/{math,filters,guidance,control,safety,utilities}` 中的可复用块** → **Library**(Linked Subsystem)。理由:细粒度复用、修一处生效、不需要独立 codegen 单元。
- **`model/<module>/shared/` 中的模块共享架构骨架(FMS Mode Manager 框架 / Controller cascade shell / Plant sensor 合成块)** → **Library**(若需要被多个 vehicle leaf 引用)或 **plain Subsystem**(若仅在本模块内单实例化)。决策权下放给 C2 / D2 / E2,但本文件给出**默认推荐:Library**(为 Phase 5 多机型复用留路径)。
- **`model/<module>/vehicles/<vehicle>/` 中的 vehicle leaf** → **Variant Subsystem**(Phase 2 仅 1 个 active variant = multicopter,但 Variant 框架就位以便 Phase 5 增加 fixwing 等)。**不**为每个 vehicle 启用一个独立的 Model Reference,见 §4.3 预算上限。

**ins_stub(harness 内)**:**plain Subsystem**(架构 v1 §6 placement,§12.3 明确"never exported to firmware")。无 firmware 契约,无独立 codegen 需求,放在 harness 中作为内部子系统即可,详细由 F2 处理。

#### 4.1.3 决策摘要表

| 资产 | 机制 | 关键理由 |
|---|---|---|
| Plant 顶层模型 | **Model Reference** | 独立 codegen,匹配 firmware lib/ drop-in;独立 build 缓存 |
| FMS 顶层模型 | **Model Reference** | 同上 |
| Controller 顶层模型 | **Model Reference** | 同上(§13 Controller 5 ms 热路径增量 build 收益尤大) |
| `model/shared/lib/*` 共享块 | Library | 细粒度复用,跨模块共享 |
| `model/<m>/shared/*` 模块共享架构 | Library(默认) / Subsystem(豁免) | 默认复用,具体由 C2/D2/E2 决定 |
| `model/<m>/vehicles/<v>/*` vehicle leaf | **Variant Subsystem** | 单一 Variant 块容纳机型选项,见 §4.3 上限 |
| `model/harness/ins_stub/*` | plain Subsystem | 仅 MIL,不 export |

### 4.2 (b) Phase 2 变体轴枚举与其他显式冻结

#### 4.2.1 唯一允许的变体轴

**唯一变体轴:vehicle class**(架构 v1 §10.3 Layer C 唯一变体维度)。

Phase 2 内该轴的枚举:**{multicopter}**(单元素,degenerate)。

Variant Subsystem 在 Phase 2 落地形式:每个 `model/<m>/vehicles/` 下的 Variant Subsystem 仅有 1 个 active variant block(multicopter)。Variant 控制变量预留命名(由 A2 命名规则定具体名),但取值集合 Phase 2 只有 `{MULTICOPTER}`。

依据:

- 架构 v1 §16 smallest slice 显式列出"explicitly excluded ... multi-vehicle support"。
- 架构 v1 §18 Phase 2 退出条件只要求"arm/hold/basic reference-tracking closed loop runs in MIL using `ins_stub`",不涉及任何 vehicle 多样性。
- 架构 v1 §19 first target 明确为"multicopter-only vertical slice"。
- 00-design-plan §5 显式排除"多旋翼以外的机型 leaf 设计"。

#### 4.2.2 其他变体轴显式冻结清单

下列**所有**潜在变体轴在 Phase 2 **冻结为单一组合**(不允许 Variant Subsystem / 编译时分支 / 参数集切换 表达这些维度):

| 维度 | Phase 2 冻结值 | 冻结理由 | Phase 5 扩展接入点(预留,不实现) |
|---|---|---|---|
| 传感器配置(IMU 数 / 是否含 magnetometer / GPS 类型) | 与多旋翼最小切片一致(`IMU_Bus + MAG_Bus + Barometer_Bus + GPS_uBlox_Bus`,§5.1.7 / §8.1) | §16 smallest slice;ins_stub 只输出单一 `INS_Out_Bus` | F1 / F2 增加可选传感器子集 |
| Mission profile | "arm → takeoff/hold → manual or position hold → land/disarm"(§16.6) | smallest slice 唯一场景 | D1 + H1 扩展 mission 集合 |
| Actuator count / 拓扑 | 多旋翼 4 旋翼默认(具体由 E4 / C3 给数,本文件不约束数值) | E4 / C3 leaf 范围;Phase 2 不引入分配矩阵参数化 | E4 在 Phase 5 增加 6 / 8 旋翼 / 倾转 |
| Mode 集合 | D1 闭环必需子集(arm/disarm/manual/altitude-hold/position-hold/takeoff/land) | D1 retain 决策 + §16 场景 | D1 / D5 扩展(含 Auto / Mission 完整集合) |
| Controller 环路启用 | E1 cmd_mask 裁剪规则下的多旋翼最小启用集 | §12.2.2 "only if controller contract really needs it";多旋翼默认全启用 | E1 / E4 在 Phase 5 按机型启用裁剪 |
| 数值精度(单 / 双) | 单一选择(由 I4 codegen 配置决定,Phase 2 不并存两套) | 防止 codegen / SIL 比对面爆炸 | 不预留(架构上不期望多精度并存) |
| 调度速率 | 固定(Plant 1ms / Controller 5ms / FMS 20ms,架构 v1 §13) | 架构 v1 §13 已定 | 不预留(速率契约稳定) |
| 坐标系 / 单位 | A2 命名规则锁定 | 单一来源 | 不预留 |

#### 4.2.3 冻结策略落地

- **不允许**通过 Variant Subsystem / Configurable Subsystem / `Variant Source` 表达上述冻结维度。
- **不允许**在参数 struct(`FMS_PARAM` / `CONTROL_PARAM` / `PLANT_PARAM`)中用一个字段开关上述维度的存在与否(可以是数值参数,但不能是结构存在性开关)。
- 任何在 Phase 2 期间提议引入第二条变体轴的行为视为契约风险,需走 INDEX 决策日志(即便 contract_impact=no,也建议登记策略偏离)。
- §17 Risk 7(Variant complexity)对应的具体防线:本节冻结清单 + §4.3 预算上限。

### 4.3 (c) 过度抽象预算上限

针对架构 v1 §17 Risk 3(Shared-vs-vehicle over-abstraction)与 Risk 7(Variant complexity),本文件设定以下**数值化硬预算**。突破任一条预算线即视为应触发 refactor 评审。

#### 4.3.1 Variant Subsystem 容量预算

| 预算项 | 上限 | 触发条件 → 处置 |
|---|---:|---|
| 单个 Variant Subsystem 内并存的 vehicle 数 | **2** | ≥ 3 vehicle 时,该 Variant 块拆分为按机型族的多个 Variant 块,或上推到 vehicle-level 独立 Model Reference |
| Variant Subsystem 嵌套深度 | **2** | 第 3 层 Variant 出现 → 必须 refactor(改用枚举参数 + 单一实现块) |
| 单个模块(Plant / FMS / Controller)内 Variant Subsystem 数量 | **5** | 第 6 个 Variant 块出现 → 触发 §4.3.4 review checkpoint |
| 单个 vehicle leaf 文件数(`model/<m>/vehicles/<v>/` 下) | **15** | 超出则该 leaf 必须按子领域(geometry / mixer / params / state-init / 其他)再分目录,但仍限 1 个 vehicle |

> Phase 2 实际值:Variant Subsystem 内 vehicle 数 = 1(multicopter only),嵌套深度 = 1,Variant 块数 ≤ 3 / 模块。预算与 Phase 2 现实之间留有充足余量,以防止过早 refactor。

#### 4.3.2 共享层抽象预算

| 预算项 | 上限 | 理由 |
|---|---:|---|
| `model/shared/lib/<sublib>/` 中单个共享块的入选机型数 | ≥ **2**(预期复用必须有) | 防止把"只有多旋翼会用"的块塞进 shared(架构 v1 §17 Risk 3 直接对应) |
| `model/<m>/shared/` 中骨架 / 模板的可参数化变量数 | **10** | 超出则该骨架可能在做"通用接口"而非"模块共享";触发 review |
| Mode / cmd_mask 的 enum 值数 | 由 B2 锁定(本文件不重复)| 跨模块影响,任何新增需走 B 区 |

#### 4.3.3 shared / leaf 切分判据(闭合 A1 §4.2 §2.4 / §10.3)

判定一个块归 shared 还是 leaf 用以下顺序检查(全部满足 → shared,否则 leaf):

1. 该块的**外部接口**(输入 / 输出 bus 或参数)是否对 ≥ 2 个 vehicle 类相同?
2. 该块的**内部数学语义**对所有候选 vehicle 是否一致(允许参数化但不允许结构性差异)?
3. 该块对应的**参数**是否能用一个统一 schema 表达(参数值可不同,但字段结构相同)?
4. 该块是否**不**依赖某个机型独有的物理量(例如 multicopter 推力轴方向)?

任一不满足 → leaf。Phase 2 由于只有 multicopter,该判据**不能根据"已观测到的复用"判**,而是**根据"已知有 ≥ 2 vehicle 类需求且接口相同"判**(避免基于唯一一个样本的过早抽象)。

豁免清单(架构 v1 §10.3 "leaf-level only" 的反例,允许在共享层但来源单机型):

- 架构 v1 §12.4 中明确归共享的 sensor 合成块(IMU / MAG / Barometer / GPS)— 即便目前只服务多旋翼 Plant,也归 `model/shared/lib/utilities` 或 Plant shared,因为接口是契约级 bus(`IMU_Bus` 等)。
- B 区契约 bus / enum / param schema — 本来就是 repo-wide 单一来源(架构 v1 §14)。

#### 4.3.4 §17 Risk 3 防线 review checkpoint

每次进入新的 vehicle 拓展(第二个机型)前,必须完成下列 checkpoint:

1. 现有 `model/shared/lib/*` 中的每个块,审查"该块是否真的对新机型有复用"(若否 → 下沉到原 vehicle leaf)。
2. 现有 `model/<m>/shared/*` 骨架中的可参数化变量数,审查是否突破 §4.3.2 预算。
3. 当前 Variant Subsystem 数量,审查是否突破 §4.3.1 预算。
4. 任何在 Phase 2 期间已经"假设第二机型会用到"的抽象,标 `tentative`,直到第二机型真正落地。

此 checkpoint 在 Phase 5 启动评审中执行。Phase 2 期间作为编辑器纪律(任何新增 shared 块时作者自检)。

### 4.4 (d) OQ1 决策记录:vehicle scope

#### 4.4.1 OQ1 原文

架构 v1 §17 开放问题 OQ1:

> "Should the first 2025b delivery target multicopter only, or multicopter plus fixed-wing shared scaffolding?"

A1 §4.3 已将此 OQ 分派给 A5 闭合。

#### 4.4.2 决策

**决策**:**Phase 2 multicopter only。fixed-wing 脚手架推迟到 Phase 5。**

依据(主从架构 v1 文本,逐条带锚):

1. 架构 v1 §16 smallest slice 第 1–6 项全部以"one multicopter ..."为限,显式排除"multi-vehicle support"。
2. 架构 v1 §19 first implementation target 明确"multicopter-only vertical slice"。
3. 架构 v1 §18 Phase 2 退出仅要求多旋翼 MIL 闭环;Phase 5 才是"vehicle expansion"。
4. 架构 v1 §17 Risk 3(over-abstraction)与本文件 §4.3 over-abstraction 预算共同建议:在只有 1 个机型样本时,**不可能**做出经过验证的"shared scaffolding"——抽象只能在第二个机型出现时被验证。在此之前任何 fixed-wing 脚手架都会成为未被使用的代码,违反 §4.3.3 判据 1。
5. 00-design-plan §5 显式范围排除"多旋翼以外的机型 leaf 设计",与本决策一致。

#### 4.4.3 在仓库树中的处置(架构 v1 §6 占位目录)

架构 v1 §6 仓库树中 `model/{plant,fms,controller}/vehicles/` 下保留了 `fixwing/`、`vtol/`、`boat/`、`car/`、`submarine/`(Controller 区)等占位目录。本文件对它们的处置:

- **设计阶段**(当前):**不创建**这些占位目录的 README 或 stub。它们仅作为架构 v1 文本中的目录承诺,不在文件系统层面物化。
- **Phase 2 实施阶段**(当前 A5 决策与之约束):仍**不创建**。`model/<m>/vehicles/` 下仅出现 `multicopter/` 一个目录加上 `template/`(架构 v1 §6 已列 `model/plant/vehicles/template/`,作为 Phase 5 创建新 vehicle 时复制起始点)。
- **Phase 5**(扩展阶段,本文件不实施):由当时的扩展工作项创建对应目录 + 装填 leaf。

理由:占位空目录在 Simulink + Git 工作流中通常需要 `.gitkeep`,而 `.gitkeep` 又会被脚本误识别。直接不物化,以"目录在 §6 文档承诺,但文件系统不创建"的方式表达"未来支持"。

#### 4.4.4 forward path(Phase 5 接入路径,仅声明,不实施)

Phase 5 扩展第二个 vehicle 时的接入步骤(放在此供 Phase 5 工作项参考,不在本工作项实现):

1. 在 `model/<m>/vehicles/` 下复制 `template/` 创建 `<vehicle>/`。
2. 将该 vehicle 加入对应模块的 Variant Subsystem 的 active variants 集(注意 §4.3.1 预算)。
3. 触发 §4.3.4 over-abstraction review checkpoint。
4. 同步更新 A2 命名规则的 vehicle 命名空间;若引入新 enum 值则走 B2 流程。
5. 增加该 vehicle 对应的 leaf 工作项(类比 C3 / D5 / E4)。

### 4.5 Controller 的 optional `Plant_States_Bus` 变体接入点(框架)

闭合 A1 §4.2 §8.4 缺口("optional `Plant_States_Bus` only in harness/HIL/SIH variants" 启用条件未定)。

本文件给框架,具体物理拓扑由 G1 处理:

- Controller 顶层模型 **不**直接暴露 `Plant_States_Bus` 输入。Controller 的契约级输入仍只有 `FMS_Out_Bus` 与 `INS_Out_Bus`(架构 v1 §8.4)。
- 在 **harness 层**(`model/harness/`),G1 设计的顶层装配中,可以在 Controller 的 referenced model **外部**包一个 harness-only Variant Subsystem,在该 variant 中提供 `Plant_States_Bus` → Controller 的旁路通道(仅 SIH / HIL variant)。
- 该 harness Variant Subsystem 的 Variant 控制变量(例如 `HARNESS_VARIANT ∈ {MIL, SIH, HIL}`)与 §4.2 vehicle 变体轴是**正交**的两条独立轴。
- `HARNESS_VARIANT` 不在 §4.2 的"冻结其他变体轴"清单中,因为它是 harness 层而非 Plant/FMS/Controller 顶层模型层;Phase 2 仅 MIL 落地,SIH / HIL 在 Phase 3 才实施。Phase 2 该 Variant Subsystem 也只 1 个 active variant(MIL)。

### 4.6 codegen 与 build 影响摘要

供 I4 / B4 参考,本节不替代它们的设计:

1. Model Reference 顶层 → 每个模块独立 `<Module>.c/.h` 产物 → 与 §5.1.10 lib/ drop-in 直接对齐。
2. Variant Subsystem(vehicle 轴)→ 推荐使用 **"Variant Sources"/"Single design"** 配置(只生成 active variant 的代码,而非 all-variants codegen),Phase 2 active variant = multicopter,生成的 `<Module>.c` 只含多旋翼 leaf。具体配置项由 I4 锁定。
3. Library(shared 块)→ inline 进消费它的 referenced model 的 `<Module>.c`(Library 不产 `.c`)。
4. harness 层 Variant Subsystem(MIL/SIH/HIL)→ 不参与 firmware export(架构 v1 §11.3 "deliberately place selected outputs into `export/firmware/<module>/<vehicle>/`",harness 不 export)。
5. 单 vehicle 切片下,Variant 控制变量虽存在但取唯一值,**不**进入 firmware-visible param struct(避免在 `*_PARAM` 中暴露 vehicle 选择字段);该控制变量是 codegen-time 而非 runtime,具体由 B3 / I4 落地。

## 5. 已知风险与悬而未决问题

- **Variant Subsystem 与 firmware 现有"single-vehicle library"模式的契合度**
  - 影响:firmware 当前期望每模块 lib/ 下放一份 `<Module>.c`,而 Variant Subsystem 在 "Variant Sources/Single design" 模式下确实只生成 active variant — 但若未来切到 "All-variants" 模式,会产生多份代码,这与 firmware 期望不符。
  - 处置:本文件 §4.6.2 锁定 "Single design" 配置;I4 在 codegen 配置中列为强制项;若未来需要 all-variants,需走契约影响评审。

- **Phase 5 第二机型出现时,§4.3.3 判据 4 中的"机型独有的物理量"边界可能模糊**
  - 影响:多旋翼推力轴方向 vs 固定翼气动力,若被错误归类到 shared,会污染 shared/lib。
  - 处置:Phase 5 启动时由 §4.3.4 review checkpoint 实际执行;Phase 2 不预判。

- **`HARNESS_VARIANT` 与 vehicle 轴正交的隐性依赖**
  - 影响:若 harness Variant 设计中无意引入对 vehicle 轴的依赖(例如 SIH variant 只在 multicopter 生效),实际上变成了二维变体。
  - 处置:G1 设计该 harness Variant 时,本文件 §4.2.3 冻结策略对 G1 也适用 — Phase 2 SIH/HIL 不实施,所以 G1 落地时 `HARNESS_VARIANT` 也是 degenerate(MIL only),自然正交。

- **Variant 控制变量命名待 A2 锁定**
  - 影响:本文件用了 `MULTICOPTER`、`HARNESS_VARIANT`、`MIL/SIH/HIL` 等示意名,A2 命名规则可能改写。
  - 处置:本文件保持示意名,A2 完成后由 A3 / G1 / C2 / D2 / E2 在引用时按 A2 规则替换;不影响本文件策略本身。

- **架构 v1 §6 中 Controller 区有 `submarine/` 而 Plant/FMS 没有**
  - 影响:架构 v1 中机型集合在 Plant/FMS/Controller 三个模块间不完全一致(submarine 仅 Controller),Phase 5 扩展时会暴露不对称。
  - 处置:本文件不修架构 v1;在 §4.4.3 处置中只关注 Phase 2 仅多旋翼,Phase 5 扩展工作项需要协调三模块的机型集合一致性。

## 6. 退出条件复核

A5 在 [`00-design-plan.md`](../00-design-plan.md) §4.A 中的退出条件原文(2026-05-05 收紧版):

> 多机型(Multi/FW/VTOL/...)的 Variant Subsystem vs 独立模型 抉择 + 多旋翼 leaf 的变体管理方案;**(a) 各顶层模型的 Model-Reference vs Library vs Subsystem 选型决定;(b) Phase 2 变体轴枚举 + 其他轴显式冻结;(c) 过度抽象预算上限(单 Variant 块最多支持的 vehicle 数 / leaf 文件数);(d) 记录架构 v1 §17 OQ1(vehicle scope)的最终选择**

逐条复核:

| # | 退出条件原文(分解) | 本文档依据 | 状态 |
|---|---|---|---|
| 1 | "多机型 ... Variant Subsystem vs 独立模型 抉择" | §4.1.2(三个顶层模型 = Model Reference;vehicle leaf = Variant Subsystem;§4.1.3 决策摘要表) | 满足 |
| 2 | "多旋翼 leaf 的变体管理方案" | §4.2.1(vehicle 轴 Phase 2 = {multicopter} 单元素 Variant)+ §4.4.3(占位目录处置) | 满足(框架级;具体 leaf 内容由 C3/D5/E4) |
| 3 | (a) 各顶层模型选型 | §4.1 全节,§4.1.2 / §4.1.3 给出 Plant / FMS / Controller 三者 = Model Reference 的决定与依据(codegen / build / debug / cross-vehicle reuse 四项均覆盖) | 满足 |
| 4 | (b) Phase 2 变体轴枚举 | §4.2.1(唯一轴 = vehicle class,枚举 {multicopter}) | 满足 |
| 5 | (b) 其他轴显式冻结 | §4.2.2(传感器 / mission / actuator count / mode / 环路 / 精度 / 速率 / 坐标系全部冻结)+ §4.2.3 落地策略 | 满足 |
| 6 | (c) 过度抽象预算上限 | §4.3.1(Variant 容量预算,单 Variant 块 ≤ 2 vehicle、嵌套 ≤ 2 层、模块内 Variant ≤ 5 个、leaf ≤ 15 文件)+ §4.3.2 共享层预算 + §4.3.3 切分判据 + §4.3.4 review checkpoint | 满足 |
| 7 | (d) OQ1 最终选择 | §4.4.2(Phase 2 multicopter only;fixed-wing 脚手架推迟到 Phase 5)+ §4.4.3 占位目录处置 + §4.4.4 forward path | 满足 |

附:架构 v1 §17 风险闭合复核(A1 分派给 A5):

| 风险 | 本文档闭合点 |
|---|---|
| §17 Risk 3 Over-abstraction | §4.3.1–§4.3.4(全节即此风险防线) |
| §17 Risk 7 Variant complexity | §4.2.2 冻结清单 + §4.3.1 容量预算 |

## 7. 下游影响

按 [`01-design-relationships.md §4.1`](../01-design-relationships.md) A1 → A5 + A2,A4,A5 → A3 出边,以及 §4.3 / §4.4 / §4.7 中各下游受影响分布:

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [A3 模块边界信号清单细化](A3-module-boundaries.md) | → | §4.1.2 三模块 = Model Reference 决定 → A3 须按此粒度组织 Inport/Outport bus 清单;§4.5 Controller 顶层不暴露 `Plant_States_Bus`(harness only) |
| [A2 命名与目录约定设计](A2-naming-conventions.md)(并行,不强前置) | (并行) | §4.2 Variant 控制变量命名空间需求(`<MODULE>_VEHICLE_VARIANT`、`HARNESS_VARIANT` 等);§4.4.3 占位目录处置 → vehicle 命名规范 |
| [C2 Plant 结构设计](../C-plant/C2-plant-structural.md) | (transitively) | §4.1.2 vehicle leaf = Variant Subsystem;§4.3.1 容量预算;§4.3.3 shared/leaf 判据 |
| [D2 FMS 结构设计](../D-fms/D2-fms-structural.md) | (transitively) | 同上 |
| [E2 Controller 结构设计](../E-controller/E2-controller-structural.md) | (transitively) | 同上 + §4.5 Controller 不直接暴露 `Plant_States_Bus`;harness 旁路只在 Variant 中 |
| [G1 MIL 顶层结构设计](../G-harness/G1-mil-toplevel.md) | (transitively) | §4.5 harness Variant Subsystem 框架(`HARNESS_VARIANT` 轴);§4.6 harness 不 export;§4.4.3 仅多旋翼 |
| [C3 Plant 多旋翼 leaf 设计](../C-plant/C3-multicopter-leaf.md) | (transitively) | §4.2.1 leaf 在 Variant 内的位置;§4.3.1 leaf 文件数预算 ≤ 15 |
| [D5 FMS 多旋翼 leaf 设计](../D-fms/D5-multicopter-leaf.md) | (transitively) | 同上 |
| [E4 Controller 多旋翼 leaf 设计](../E-controller/E4-multicopter-leaf.md) | (transitively) | 同上 |
| [I4 Codegen 配置脚本设计](../I-tooling/I4-codegen-config.md) | (transitively) | §4.6 codegen 影响摘要,尤其 Variant Subsystem "Single design" 配置 |
| [B4 契约 diff 策略](../B-contracts/B4-contract-diff.md) | (transitively) | §4.1.2 / §4.6.1 三模块独立 codegen 单元 → diff 边界对齐 |

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-05 | A5 author | 初稿 |

## Self-check

- [x] frontmatter 完整,字段值合法
- [x] 上游文档全部存在且 status ≥ reviewed(架构 v1 已发布并锚定 2026-05-05 修订;A1 reviewed 2026-05-05)
- [x] 退出条件逐条复核完成,每条均给出依据(§6 表)
- [x] 引用路径全部可点击访问(架构 v1 / A1 / 00-design-plan / 01-design-relationships / 各下游 .md 路径均按 RULES §4 规则)
- [x] 不存在 RULES §5 禁则中的内容(无 .slx 截图;无可执行 .m;不复述 firmware 实现细节;不重定义 bus/enum/参数;占位目录决策与 §4.4.3 锚到架构 v1 §6 + Phase 2/5 决策日期)
- [x] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记(本文件 contract_impact=no:Variant 控制变量为 codegen-time 非 runtime,不进入 `*_PARAM`/`*_EXPORT`/bus 字段;Model Reference 的选用本身不改变 firmware 可见符号 — `<Module>_init`/`<Module>_step`/`<MODULE>_EXPORT`/`<MODULE>_PARAM` 在 §5.1 已契约化,本文件只是选用符合该契约的封装机制。N/A)
- [x] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径(本文件未镜像任何 firmware 契约,N/A)
- [x] 下游影响已沿关系图识别完毕(§7 含 A3 直接 + C2/D2/E2/G1/C3/D5/E4/I4/B4 间接下游)
- [x] 文档不超出本工作项范围(无越权设计:vehicle leaf 内部内容显式 §2 不在范围;harness 物理拓扑交 G1;codegen 配置项交 I4;bus/enum/param 字段交 B 区)
