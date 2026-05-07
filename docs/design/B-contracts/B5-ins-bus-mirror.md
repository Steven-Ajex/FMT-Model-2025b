---
work_item: B5
title: INS_Out_Bus 镜像同步流程
upstream: ["架构 v1", "A1", "A3", "A6", "A7", "B1", "B2", "B3"]
contract_impact: yes
status: reviewed
authored_at: 2026-05-07
last_reviewed_at: 2026-05-07
reviewer_verdict: pass
---

# B5 INS_Out_Bus 镜像同步流程

## 1. 目的

为模型仓**唯一不拥有的契约 bus** `INS_Out_Bus` 落定**跨仓 firmware → 模型仓单向镜像流程**:声明 canonical 来源(firmware 单向 mirror,闭合 [架构 v1 §17 OQ6](../../architecture/2026-05-05-fmt-model-architecture-v1.md));指定抽取脚本(由 [I2](../I-tooling/I2-bus-enum-mirror.md) 实现)的 I/O 契约与版本标记策略;锁定镜像产物的存放布局与 firmware commit hash + 文件相对路径记录义务(per RULES §5);定义 PR 工作流、drift 检测与升级响应,并把这些机制映射到 [架构 v1 §17 risk 2](../../architecture/2026-05-05-fmt-model-architecture-v1.md)("`INS_Out_Bus` contract drift")的 detect / recover / escalate 三段。

## 2. 范围

**在范围:**

- **canonical 来源声明**(§4.1):三选项(A firmware-canonical 单向 mirror / B 共享子模块 / C 共享 schema repo + codegen)中选定 **Option A**,记录决策依据与权衡,闭合架构 v1 §17 OQ6
- **镜像方向与频次**(§4.2):firmware → 模型仓单向;trigger = 任何对 `FMT-Firmware/src/model/ins/<variant>/lib/INS_types.h` 的 firmware commit;cadence = opportunistic + monthly drift sweep
- **抽取脚本 I/O 契约**(§4.3):**作为 [I2](../I-tooling/I2-bus-enum-mirror.md) 设计输入**给定输入参数(firmware 仓路径 + commit SHA)、输出产物路径与扩展名(`.bus` Simulink Bus 对象 mat-file 或 `.yaml` schema 描述,本文件选 `.yaml` 作为 canonical interchange,见 §4.3.2)与 header block 必备字段(`firmware_commit_sha` 40-char、`firmware_path`、`extracted_at` ISO 日期、`extracted_by` CI/人工 + bot ID)
- **版本标记策略**(§4.4):每个镜像产物携带 `schema_version` 整数;凡 firmware 有任何字段变化必须 bump、不复用旧值;G3 logging manifest 与 [B3 §4.7.5 model_info[]](B3-parameter-schema.md) 同步记录该 `schema_version`
- **镜像产物存放布局**(§4.5):`model/shared/bus/ins/` 子目录结构(`INS_Out_Bus.<ext>` + `INS_Out_Bus.firmware-source.md` + 可选 `_history/`)
- **PR 工作流**(§4.6):bot-authored mirror PR 的产出物清单、reviewer checklist、merge gate(I3 contract diff 必须 pass)
- **失败 / drift 响应**(§4.7):drift 检测的硬失败语义、>14 天未合并升级到 MIL/SIH 冻结
- **F-area 下游耦合声明**(§4.8):F1 / F2 / F3 消费侧依赖关系
- **RULES §5 合规义务**(§4.9):每个镜像产物嵌入 `firmware_commit_sha` + `firmware_path` 的 header block 强制要求与 CI 拒收策略
- **架构 v1 §17 risk 2 闭合映射**(§4.10):detect / recover / escalate 三段映射

**不在范围(由其他工作项处理):**

- `INS_Out_Bus` 字段级 schema(字段名 / 类型 / 字节宽度 / 累计偏移 / 单位 / 坐标系 / Validity 守卫)— 由 [B1 §4.3.1](B1-bus-inventory.md) 落地;B5 仅引用并标"由 B5 镜像 firmware 实际产物时 byte-verify"
- `INS_Status` / `INS_Flag` 位 / 数值定义 — 由 [B2 §4.4.10 / §4.4.11](B2-enum-inventory.md) 落地;B5 不重述
- 抽取脚本**实现细节**(MATLAB 函数签名、解析正则、错误处理、退出码、CLI 输出格式)— 由 [I2 Bus/enum 镜像脚本设计](../I-tooling/I2-bus-enum-mirror.md) 处理(同 batch / Wave 5);B5 仅给 I/O 契约
- 契约 diff 比对算法、字段顺序检查、enum 数值检查、报告输出格式 — 由 [B4 契约 diff 策略](B4-contract-diff.md) + [I3 契约 diff 脚本设计](../I-tooling/I3-contract-diff.md) 落地;B5 仅在 §4.6 / §4.7 引用 I3 作为 merge gate
- INS 内部 EKF / 估计器结构、INS 参数(`INS_PARAM`)— firmware 拥有(架构 v1 §3 / §12.3);[B3](B3-parameter-schema.md) 已确认模型仓**不**产 `INS_PARAM`(B3 §4.1 总览)
- INS_Out_Bus 消费侧 fallback / validity 处理规则 — 由 [F3 INS_Out_Bus 消费规则](../F-ins-contract/F3-consumption-rules.md) 落地;B5 仅声明 F-area 下游耦合
- 模型仓 ins_stub 的功能 / 结构 — 由 [F1](../F-ins-contract/F1-ins-stub-functional.md) / [F2](../F-ins-contract/F2-ins-stub-structural.md) 落地;B5 仅约束 ins_stub 输出 bus 必须**严格符合本镜像产物 schema**
- INS 在 SIH/HIL 阶段的真实 firmware 行为对比 — 由架构 v1 §15.3 + 后续 H 区设计落地

## 3. 依赖

| 上游 | 引用位置 | 用途 |
|---|---|---|
| 架构 v1 §3 / §5.1.8 / §10.4 / §12.3 / §17 OQ6 / §17 risk 2 | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | INS firmware-owned 决策(§3);"INS_Out_Bus 必须从 firmware 镜像、变更需 lockstep"(§5.1.8 第 8 项);"无 INS layer / harness ins_stub 是唯一模型仓 INS 产物"(§10.4);INS 不在模型仓(§12.3);OQ6 "schema versioning 在何处"由本工作项闭合;risk 2 contract drift mitigation 在本工作项落地 |
| [A1 架构 v1 评审与缺口闭合](../A-architecture/A1-v1-review.md) (status: reviewed, 2026-05-05) | A1 §5.1.8 / OQ6 闭合责任分配 | A1 把"`INS_Out_Bus` schema 镜像位置 / 流程"列为 OQ6,指派给 B5 闭合;本文件 §4.1 即为该闭合 |
| [A3 模块边界信号清单细化](../A-architecture/A3-module-boundaries.md) (status: reviewed, 2026-05-07) | A3 §4.4.4(FMS 消费侧)+ §4.5.3(Controller 消费侧)+ §4.8(INS 消费 dependency 矩阵) | INS_Out_Bus 在模型仓侧的消费方依赖矩阵;A3 §4.4.4 显式声明"完整字段表由 [B5] 在镜像产物中锁定";本文件 §4.5 镜像产物即为该锁定载体 |
| [A6 Init/Reset 状态契约](../A-architecture/A6-init-reset-contract.md) (status: reviewed, 2026-05-05) | A6 §4.5.3 INS gate | INS 消费方在 init / reset 时刻的 gate 行为;B5 镜像产物保留 `INS_Status.ready` 与 `INS_Flag.*` 位以支撑 A6 §4.5.3 行为(本文件不重述,F3 落 fallback 行为) |
| [A7 时间与时间戳约定](../A-architecture/A7-time-conventions.md) (status: reviewed, 2026-05-05) | A7 §4.1 / §4.6 | `INS_Out_Bus.timestamp` 字段单位 / 编码与全局时间约定一致(`uint32` ms,firmware-legacy 无后缀);B5 镜像产物对该字段不做特殊处置,沿 A7 通用约定 |
| [B1 Bus 清单与 schema 设计](B1-bus-inventory.md) (status: reviewed, 2026-05-07) | B1 §4.3.1(`INS_Out_Bus` 消费侧 best-effort schema)+ §4.6.5 / §4.6.6(`INS_Status` / `INS_Flag` 子 schema)+ §4.9(B1 ↔ B2 / B3 交叉引用)+ §4.11(sample-time 覆盖) | B1 给消费侧推断字段顺序;B5 镜像产物**取代** B1 §4.3.1 的最佳推断作为 firmware-canonical 顺序;B5 不重定义字段表,引用 B1 §4.3.1 作为预期消费形态 |
| [B2 Enum 清单与数值锁定](B2-enum-inventory.md) (status: reviewed) | B2 §4.4.10 `INS_Status` + §4.4.11 `INS_Flag` | INS 消费用到的 enum / bitfield 数值定义;B5 镜像产物中 `INS_Status` / `INS_Flag` 字段**类型**保持与 B2 一致(`uint32` bitfield) |
| [B3 Parameter schema 设计](B3-parameter-schema.md) (status: reviewed) | B3 §4.1 总览("不存在 INS_PARAM")+ §4.7.5 model_info[] ledger | 确认模型仓不产 `INS_PARAM`(`INS_PARAM` 由 firmware 拥有,与 B5 的"firmware-canonical"决定一致);本文件 §4.4 `schema_version` 同步入 G3 manifest 与 EXPORT model_info[] 时,与 B3 §4.7.5 `schema_version` 字段语义对齐(注:B3 的 `schema_version` 指模型仓 EXPORT 自身的 schema 版本;本文件指 INS_Out_Bus 镜像产物的 schema 版本,二者**独立**计数,见 §4.4.4) |
| **co-seal Wave 5 同期 sibling**:[B4 契约 diff 策略](B4-contract-diff.md) *(drafting in flight)*、[I2 Bus/enum 镜像脚本设计](../I-tooling/I2-bus-enum-mirror.md) *(drafting in flight)*、[I3 契约 diff 脚本设计](../I-tooling/I3-contract-diff.md) *(drafting in flight)* | per [01-design-relationships.md §6 Wave 5](../01-design-relationships.md) | **B5 不与 I2/I3 同 batch co-seal**(per 任务说明);但允许**前向引用** I2 作为抽取脚本设计的承接点、I3 作为 merge gate 的契约 diff 实现承接点;B5 给 I2/I3 提供 I/O 契约与触发时机要求 |
| firmware INS 头文件 | `FMT-Firmware/src/model/ins/<variant>/lib/INS_types.h` @ `FMT-Firmware @ <pending hash>`(commit hash 在 INDEX 决策日志登记本工作项时补齐,与 [A3 §3](../A-architecture/A3-module-boundaries.md) / [A6 §3](../A-architecture/A6-init-reset-contract.md) / [A7 §3](../A-architecture/A7-time-conventions.md) / [B1 §3](B1-bus-inventory.md) / [B2 §3](B2-enum-inventory.md) / [B3 §3](B3-parameter-schema.md) 同 batch 占位约定;FMT-Firmware 仓未挂载在本环境)| `INS_Out_Bus` 字段顺序 / 类型的 firmware 端权威来源;**本文件不重述字段表**,只通过镜像流程把 firmware 的字段顺序作为 canonical 引入模型仓。`<variant>` 子目录(例如 `multicopter` / `fixwing` / 或单一 `ins/lib/`)在本环境无法 ground-truth;§4.5.2 给路径约定与 unknown-variant 处置 |

注:本工作项 contract_impact=yes;**直接定义模型仓侧消费 firmware 契约的镜像流程**,变更必须经 INDEX 决策日志登记(per RULES §10)。

## 4. 设计内容

### 4.1 Canonical 来源声明(闭合架构 v1 §17 OQ6)

#### 4.1.1 三选项对照

| 选项 | 描述 | 优势 | 劣势 |
|---|---|---|---|
| **A** firmware-canonical, model-repo 单向 mirror | INS_Out_Bus 字段顺序 / 类型 / enum 位号的**唯一权威**是 `FMT-Firmware/src/model/ins/<variant>/lib/INS_types.h`;模型仓持有镜像产物 `model/shared/bus/ins/INS_Out_Bus.yaml`;每次 firmware tip 改动通过 bot-authored PR 自动同步到模型仓;CI 拒绝任何模型仓导出在 firmware 已变更但镜像未更新时通过 | 协同开销最低、与架构 v1 §3 / §5.1.8 "INS 由 firmware 拥有"决定**完全一致**;不需要新仓库或子模块;镜像产物 review 在模型仓 PR 内完成;CI gate 只需 contract diff(I3) | 依赖 firmware 仓在 PR 阶段把 `INS_types.h` 变更通知到模型仓 bot(可能漏触发);bot 实现成本(由 I2 落地);"firmware tip"语义需明确(§4.2.3) |
| **B** 共享 git submodule | 把 `INS_Out_Bus` schema 抽到独立仓 `fmt-ins-contract`,作为 git submodule 在 firmware 与 model 仓内引用 | 单点真理在第三方仓;两侧都"读"同一文件 | 引入第三方仓与对应权限治理;submodule 操作流程在 model 仓与 firmware 仓的工程团队学习成本;codegen 流程需要配合;不解决"什么时候同步"问题(submodule 仍需手动 bump) |
| **C** 共享 YAML / JSON Schema 仓 + 双侧 codegen | 上游 schema 在共享仓,firmware 与 model 都从该 schema 生成 `*_types.h` / Simulink Bus 对象 | 形式上最对称;两侧不再有"谁是真理" | 与 firmware 现有 hand-written `INS_types.h`(架构 v1 §3 / §12.3 native C/C++)冲突 — firmware 现状是估计器堆栈与传感器 driver 紧耦合;codegen 入侵 firmware 是不可接受的范围扩张;迁移成本远超价值 |

#### 4.1.2 选定:**Option A**(firmware-canonical, model-repo 单向 mirror)

**决策依据:**

1. **与架构 v1 §3 / §5.1.8 / §12.3 完全一致** — INS 在 firmware 是 hand-written native C/C++(PX4 ECL-based),与传感器 driver 紧耦合;模型仓**不**生成 INS、**不**导出 INS、**不**拥有 INS_PARAM([B3 §4.1](B3-parameter-schema.md) 已确认)。Option A 把这一架构选择固化到镜像流程,不引入新仓库 / 新治理边界。
2. **协同开销最低** — Option B 的 submodule 与 Option C 的 codegen 都引入"两侧均需修改"成本;Option A 把所有变更责任集中在 firmware 一侧,模型仓只承担 mirror review。
3. **CI gate 简单可实现** — Option A 只需 (a) bot 检测 firmware tip 改动并产出 mirror PR(由 [I2](../I-tooling/I2-bus-enum-mirror.md) 落地)+ (b) contract diff 在模型仓导出前 hard-fail(由 [I3](../I-tooling/I3-contract-diff.md) + [B4](B4-contract-diff.md) 落地)。两者都是模型仓侧工具,不入侵 firmware。
4. **架构 v1 §17 risk 2 mitigation 路径清晰** — risk 2 ("INS_Out_Bus contract drift") 的 mitigation 已声明为"contract-diff 脚本在每次导出比对模型仓镜像与 firmware";Option A 直接对应该 mitigation;Option B/C 需要额外的 schema-merge 路径。

**权衡(已知 trade-off):**

- Option A 把"firmware 已动但 mirror 未跟上"的窗口期暴露在模型仓侧 — §4.7 失败响应规则(drift detection + 14-day MIL freeze)正是为闭合此窗口期。
- Option A 不解决"firmware 与 model 互相独立 review"的根本对称性问题;但架构 v1 已显式拒绝对称性(§3 INS firmware-owned,non-goal §3 第 5 项),故此非缺陷而是设计意图。
- 若未来 INS 模块从 firmware 迁回模型仓(架构 v1 §17 OQ3 已 resolved 为"INS 留 firmware";假定不变),需重启本决策。

#### 4.1.3 OQ6 闭合声明

> 架构 v1 §17 OQ6:"Where does `INS_Out_Bus` schema versioning live — in the model repo, in firmware, or in a shared submodule? Need a single source of truth and a diff workflow."

**闭合**:single source of truth = **firmware**(`FMT-Firmware/src/model/ins/<variant>/lib/INS_types.h`);schema versioning = `schema_version` 整数(§4.4),由 firmware 一侧 bump,模型仓 mirror 同步;diff workflow = bot-authored mirror PR(§4.6)+ contract diff hard-fail(§4.7,由 I3 实现)。本工作项即 OQ6 的闭合载体。orchestrator 在 INDEX 决策日志登记本工作项时同步标注 OQ6 已 resolved(per RULES §6 self-check 第 6 项的 contract_impact=yes 流程)。

### 4.2 镜像方向与频次

#### 4.2.1 方向

**严格单向**:`firmware → model 仓`。

- firmware 不读模型仓任何 INS 文件(模型仓侧没有 INS_PARAM,没有 INS Simulink 模型,没有 INS 导出产物 — 与架构 v1 §3 / §5.1.5 / §10.4 一致)。
- 模型仓侧任何对 `INS_Out_Bus` 字段表的"修改提议"必须**先反映到 firmware** `INS_types.h`,再由 bot 自动镜像到模型仓 — 模型仓**不**接受跳过 firmware 直接编辑 mirror artifact 的 PR(CI gate,§4.6 reviewer checklist)。

#### 4.2.2 触发条件(trigger)

镜像 PR 的触发器是**任何对 `FMT-Firmware/src/model/ins/<variant>/lib/INS_types.h` 的 commit**(包括但不限于:字段增删、字段顺序调整、字段类型变更、`INS_Status` / `INS_Flag` 位号变更)。

- 不触发:对 firmware 估计器**实现** `.c` / `.cpp` 文件的修改(架构 v1 §3 / §12.3 — 估计器算法在 firmware 内部演化,不影响契约 bus)。
- 不触发:对 firmware 其他模块 `*_types.h` 的修改(模型仓侧 Plant/FMS/Controller 的契约 bus 由模型仓**自身导出**,不走 mirror 流程)。
- 不触发:firmware `<variant>` 目录下其他文件(例如 sensor driver header)。

实现侧:bot 通过 firmware repo 的 webhook 或 cron-poll 监听 `INS_types.h` 的 SHA 变化(实现细节由 I2 落地)。

#### 4.2.3 "firmware tip" 语义

**定义**:firmware 的 `master` (或后续约定的 release 分支)的 latest commit。

- mirror PR 一律针对 firmware `master` tip 产出;feature 分支变更不触发 mirror。
- 当 firmware 同时存在多个 release 分支时(架构 v1 暂未定义此情况,见 §5 风险),本工作项默认只跟踪 `master`;多分支 tracking 由后续修订处理。

#### 4.2.4 Cadence(频次)

| 类别 | 触发 | 期望响应窗口 |
|---|---|---|
| **Opportunistic**(每 firmware PR 触发) | bot 检测到 `INS_types.h` 改动 | bot 在 24 小时内开 mirror PR;reviewer 在 7 天内 merge |
| **Scheduled monthly drift sweep** | 每月第一个工作日 bot 主动比对 firmware tip 与模型仓 mirror artifact 的 `firmware_commit_sha` 字段 | 若发现 mirror 落后 ≥ 1 commit 但 opportunistic PR 未触发(漏触发场景),bot 补开 PR |

monthly sweep 是 opportunistic 的安全网,覆盖 webhook 漏触发或仓库迁移导致的 silent drift。具体调度由 [I2](../I-tooling/I2-bus-enum-mirror.md) + 模型仓 CI 配置(本工作项不涉及 CI 平台选型)。

#### 4.2.5 Owner side(谁负责什么)

| 角色 | 责任 |
|---|---|
| **firmware 维护者** | 对 `INS_types.h` 任何修改在 firmware 仓 PR 描述中声明字段变化;**不**需要主动通知模型仓(bot 自动检测);若 bump `INS_Status` / `INS_Flag` 位号,在 commit message 中以约定格式标注以利 mirror PR 自动 summary 生成 |
| **model 仓 mirror bot**(由 [I2](../I-tooling/I2-bus-enum-mirror.md) 落地) | 检测 trigger → 运行抽取脚本 → 产出新 mirror artifact + `firmware-source.md` → 开 PR;PR 标题 / body 按约定模板(§4.6.1) |
| **model 仓 reviewer**(人工) | 按 §4.6.2 checklist 审 mirror PR;特别关注 `INS_Status` / `INS_Flag` 位增量、bus 总宽度变化、字段顺序变化;若发现非预期变化,先与 firmware 维护者沟通再决定 merge / reject |
| **orchestrator** | mirror PR merge 后,在 INDEX 决策日志登记 firmware commit hash 与 mirror 产物 schema_version |

### 4.3 抽取脚本 I/O 契约(给 [I2](../I-tooling/I2-bus-enum-mirror.md) 的设计输入)

本节**不**设计抽取脚本的实现(MATLAB 函数签名、解析正则、错误处理、退出码均由 [I2](../I-tooling/I2-bus-enum-mirror.md) 落地);只锁定抽取脚本必须满足的 **I/O 契约**。

#### 4.3.1 输入

| 参数 | 类型 | 必填 | 含义 |
|---|---|---|---|
| `firmware_repo_path` | filesystem path (字符串) | yes | firmware 仓 checkout 的本地路径(bot 在 CI 环境提前 clone) |
| `firmware_commit_sha` | 40-char SHA1 字符串 | yes | 抽取目标 commit;不传或非 40-char 时脚本必须 hard-fail |
| `firmware_variant` | 字符串(`multicopter` / `fixwing` / `<empty>` 等) | no | 选择 `<variant>` 子目录;`<empty>` 表示 firmware 在 `ins/lib/` 直接放统一头文件(本环境无法 ground-truth,见 §5 风险) |
| `output_dir` | filesystem path | yes | mirror artifact 写入目录(典型 `<model_repo>/model/shared/bus/ins/`) |

#### 4.3.2 输出

抽取脚本必须产出**两个文件**:

1. **`INS_Out_Bus.yaml`**(canonical mirror artifact;扩展名选定见 §4.3.3)
   - 内容:`INS_Out_Bus` 完整字段表(field name / Simulink type / 字节宽度 / 累计 offset / 单位 / 坐标系 / encoding / validity gate)+ `INS_Status` 子 schema + `INS_Flag` 子 schema
   - 顶部 header block(必备字段见 §4.3.4)
   - 由 Simulink 端通过 [I1 init 脚本](../I-tooling/I1-init-script.md) 在加载时把 YAML 转 `Simulink.Bus` 对象(本工作项不设计该转换路径,由 I1 + I2 协同)

2. **`INS_Out_Bus.firmware-source.md`**(人类可读 source ledger)
   - 记录:firmware commit hash、firmware 文件相对路径、commit 时间、commit message 的 INS 相关 summary、本次镜像产物相对上次的字段差异列表(如有)
   - 该文件是 reviewer 在 PR 中的主要审阅对象之一

#### 4.3.3 输出扩展名选定:`.yaml`(canonical)

权衡:

| 选项 | 优势 | 劣势 |
|---|---|---|
| `.bus` mat-file(Simulink Bus 对象序列化) | Simulink 加载零转换 | 二进制 — git diff 不可读;reviewer 无法在 PR 中逐字段比对 |
| `.yaml` schema 描述 | 文本 — git diff 直观;reviewer 可逐字段比对;跨工具友好 | 需要 Simulink 端 loader(由 I1 落地) |
| `.json` | 同 yaml | 注释不友好(reviewer 加 inline 解释难) |

**选定 `.yaml`**:reviewer 体验优先(mirror PR 的核心价值是人审 + diff 自动化);Simulink 端 loader 由 I1 一次性投入。这与架构 v1 §14.1 "Maintain bus definitions from one canonical source" 的精神一致 — canonical source 是文本而非二进制。

#### 4.3.4 Header block 必备字段(强制)

`INS_Out_Bus.yaml` 顶部必须含以下 YAML header block(注释或 metadata 节):

```yaml
# INS_Out_Bus mirror artifact — DO NOT EDIT BY HAND
# Generated by I2 extraction script from FMT-Firmware
metadata:
  firmware_commit_sha: <40-char SHA1>           # 必须;非空;非 40-char 时 CI 拒收 (per §4.9)
  firmware_path: src/model/ins/<variant>/lib/INS_types.h  # 必须;非空;CI 校验路径前缀
  extracted_at: <ISO 8601 date, e.g. 2026-05-07T14:30:00Z>  # 必须
  extracted_by: <CI runner ID 或 human + bot ID>  # 必须;典型 "ci-bot@github-actions" 或 "user@laptop + i2-script-v0.3"
  schema_version: <integer, monotonically increasing>  # 必须;每次 firmware 字段变化 bump (per §4.4)
  i2_script_version: <semver, e.g. 0.3.0>  # 抽取脚本自身版本,便于追溯解析逻辑变更
```

`INS_Out_Bus.firmware-source.md` 必须把同样信息以 markdown 表格形式重复(冗余但便于人读)。

#### 4.3.5 失败语义

抽取脚本必须 hard-fail 而**不**产出部分文件的情形:

- `firmware_commit_sha` 非 40-char 或不存在于 firmware 仓
- `firmware_path` 文件在指定 commit 不存在
- 解析 `INS_types.h` 失败(无法识别 struct 边界)
- `INS_Status` / `INS_Flag` 位号定义无法解析

每种失败必须返回非 0 退出码(具体码值由 I2 定义);CI 检测到非 0 退出即停止 PR 创建。

### 4.4 版本标记策略

#### 4.4.1 `schema_version` 字段

每个 mirror artifact 携带 `schema_version` 整数,记录于 `INS_Out_Bus.yaml` header block(§4.3.4)与 `INS_Out_Bus.firmware-source.md`。

#### 4.4.2 Bump 规则

| 触发 | bump 动作 |
|---|---|
| firmware `INS_Out_Bus` 任一字段类型 / 顺序 / 名称变更 | `schema_version` += 1 |
| firmware `INS_Status` 任一位增 / 删 / 重命名 | `schema_version` += 1 |
| firmware `INS_Flag` 任一位增 / 删 / 重命名 | `schema_version` += 1 |
| firmware `INS_Out_Bus` 字节宽度变化(包括 padding) | `schema_version` += 1 |
| firmware 其他改动(不影响 contract 位 / 字段) | 不 bump |

**禁止重用旧 `schema_version` 值**:即便 firmware 在 vN+1 后回滚到 vN-1 等价,也必须 bump 到 vN+2(单调递增)。这避免历史 mirror artifact 在 git blame / regression baseline 中混淆。

#### 4.4.3 起点

模型仓首次创建 mirror artifact 时,`schema_version = 1`。该决策本身记录于 INDEX 决策日志(per RULES §6 self-check 第 6 项)。

#### 4.4.4 与 [B3 §4.7.5 model_info[]](B3-parameter-schema.md) `schema_version` 的区分

B3 §4.7.5 中 `model_info[]` 也含 `schema_version` 字段 — 那是**模型仓 EXPORT 自身的 schema 版本**(`PLANT_EXPORT` / `FMS_EXPORT` / `CONTROL_EXPORT` 的字段集合版本)。本工作项的 `schema_version` 是 **`INS_Out_Bus` mirror artifact 的版本**,**两者独立计数**:

- 模型仓 EXPORT schema_version 由 [B3](B3-parameter-schema.md) / [I4](../I-tooling/I4-codegen-config.md) 治理
- INS_Out_Bus mirror schema_version 由本工作项治理(`model/shared/bus/ins/INS_Out_Bus.yaml` header)

为避免混淆,本工作项的 `schema_version` 在所有引用位置(G3 manifest、I3 diff 报告等)显式带前缀 `ins_bus_schema_version`。

#### 4.4.5 G3 manifest 同步记录

模型仓的运行时 manifest(由 [G3 日志与可观测性](../G-harness/G3-logging.md) 落地)必须把 `ins_bus_schema_version` 作为 run metadata 的一项(其他项含 `firmware_commit_sha` / `extracted_at` 拷贝)。这让每次 MIL run 都可以追溯到 INS_Out_Bus 的具体 firmware 来源(per RULES §5 镜像契约必须记录来源版本)。

### 4.5 镜像产物存放布局

#### 4.5.1 目录结构

```text
model/shared/bus/ins/
├── INS_Out_Bus.yaml               # canonical mirror artifact (§4.3.2/§4.3.3/§4.3.4)
├── INS_Out_Bus.firmware-source.md # 人类可读 source ledger (§4.3.2/§4.5.3)
└── _history/                      # 可选:超期 mirror artifact 保留区 (§4.5.4)
    ├── INS_Out_Bus.v1.yaml
    ├── INS_Out_Bus.v2.yaml
    └── ...
```

该路径与架构 v1 §6 推荐目录树("`model/shared/bus/ins/` # contract-only: INS_Out_Bus schema mirrored from firmware")完全一致。

#### 4.5.2 `<variant>` 子目录处置

firmware INS_types.h 的 `<variant>` 子目录(可能是 `multicopter` / `fixwing` / 单一 `ins/lib/` 等)在本环境**无法 ground-truth**(FMT-Firmware 仓未挂载)。处置:

- mirror artifact `INS_Out_Bus.yaml` header block 必须**完整记录** `firmware_path`(包含实际 variant 路径),例如 `src/model/ins/multicopter/lib/INS_types.h` 或 `src/model/ins/lib/INS_types.h`;
- 模型仓侧**单一** `INS_Out_Bus.yaml` 文件(对应模型仓 Phase 2 多旋翼 only,per 架构 v1 §16 / §19);
- 未来若 firmware 多 variant 的 `INS_types.h` 字段不一致(即不同机型 INS_Out_Bus 不同),由本文件变更日志 + 后续修订处理(目前 best-effort 假设所有 variant 共享单一 `INS_Out_Bus` schema,本风险列入 §5)。

#### 4.5.3 `INS_Out_Bus.firmware-source.md` 必备字段

```markdown
# INS_Out_Bus firmware source ledger

| 项 | 值 |
|---|---|
| firmware_commit_sha | <40-char SHA1> |
| firmware_path | src/model/ins/<variant>/lib/INS_types.h |
| firmware_commit_date | <ISO 8601> |
| firmware_commit_summary | <commit message 的 INS 相关行,bot 自动摘抄> |
| schema_version | <integer> |
| extracted_at | <ISO 8601> |
| extracted_by | <ID> |

## 字段差异(相对上一次镜像)

- <bot 自动列出 added / removed / type-changed / order-changed 字段;若首次镜像,标 "initial"> 
```

reviewer 在 PR 审阅时主要看本文件 + `INS_Out_Bus.yaml` 的 git diff 两件事。

#### 4.5.4 `_history/` 目录

可选保留超期 mirror artifact(按 `INS_Out_Bus.v<N>.yaml` 命名),便于 git blame trace 与历史 MIL run 的 schema 复现。具体保留策略(只保留 last N、按时间窗口、还是全部保留)由后续工程实践决定;本工作项**不**强制 `_history/` 必须存在,允许 mirror PR merge 时只更新 `INS_Out_Bus.yaml`(git history 已是 implicit `_history/`)。

#### 4.5.5 路径 vs Simulink data dictionary 关系

`INS_Out_Bus.yaml` 是**canonical 持久化形式**;Simulink workspace 在 [I1 init 脚本](../I-tooling/I1-init-script.md) 加载时把它转成 `Simulink.Bus` 对象 `INS_Out_Bus`(per [B1 §4.3.1](B1-bus-inventory.md) Simulink Bus Object 名)。该转换由 I1 落地,本工作项不涉及实现细节。

### 4.6 PR 工作流

#### 4.6.1 mirror PR 产出物清单

bot 检测到 trigger(§4.2.2)→ 在模型仓产出一个 PR,内容必须包含且仅包含:

1. 修改后的 `model/shared/bus/ins/INS_Out_Bus.yaml`
2. 修改后的 `model/shared/bus/ins/INS_Out_Bus.firmware-source.md`
3. 可选:新增 `model/shared/bus/ins/_history/INS_Out_Bus.v<old>.yaml`(若约定保留历史)
4. PR 标题模板:`[mirror] INS_Out_Bus → schema_version <new> (firmware <short SHA>)`
5. PR body 模板:bot 自动列出字段差异 summary、firmware commit message、`ins_bus_schema_version` 新值

mirror PR **不应**包含其他文件修改(包含其他修改 = CI hard-fail,以避免镜像 PR 与无关变更耦合)。

#### 4.6.2 Reviewer checklist(强制)

reviewer 必须逐条勾选(checklist 落到 PR template):

- [ ] `firmware_commit_sha` 是 40-char SHA1,且对应 firmware 仓 `master` 上确实存在的 commit
- [ ] `firmware_path` 字段值是 `src/model/ins/...types.h` 形式,与 firmware 实际目录一致
- [ ] `INS_Out_Bus.yaml` 中所有字段有 type / width / unit / frame / encoding(空字段标 `N/A`)
- [ ] 字段顺序变化是否预期(对比 firmware commit message + ledger 字段差异)
- [ ] `INS_Status` / `INS_Flag` 新增位是否在 firmware commit message 有解释;若新增位,需在模型仓侧 [B2](B2-enum-inventory.md) §4.4.10 / §4.4.11 同步 PR(由 reviewer 触发)
- [ ] `schema_version` 已 bump(§4.4.2 规则);非 bump = reject
- [ ] PR 仅含 §4.6.1 文件清单中允许的文件
- [ ] CI(I3 contract diff)pass

#### 4.6.3 Merge gate

**Merge 必须满足**:

1. 上述 reviewer checklist 全部勾选
2. CI 上 [I3 契约 diff 脚本](../I-tooling/I3-contract-diff.md)(per [B4](B4-contract-diff.md) coverage)对**新** `INS_Out_Bus.yaml` 与 firmware `INS_types.h` @ `firmware_commit_sha` 比对**完全一致**(零 diff)
3. CI 拒绝任何 header block 缺字段的 PR(per §4.9)

merge 后 orchestrator 在 INDEX 决策日志登记 `firmware_commit_sha` + `ins_bus_schema_version`。

#### 4.6.4 与 [I2](../I-tooling/I2-bus-enum-mirror.md) 协议

bot 是 [I2](../I-tooling/I2-bus-enum-mirror.md) 的 CI runner 实例化产物;本工作项的 §4.6.1 / §4.6.2 是 I2 必须实现的产出契约。I2 设计文档承接 §4.6 的全部细节并落 implementation。

### 4.7 失败 / drift 响应

#### 4.7.1 Drift 检测(detect)

**定义**:firmware tip 已变更但模型仓 mirror artifact 未对应 bump,即:
- `model/shared/bus/ins/INS_Out_Bus.yaml` header `firmware_commit_sha` ≠ firmware tip SHA(at the time of contract diff run)

**检测方:** [I3 契约 diff 脚本](../I-tooling/I3-contract-diff.md)(per B4 coverage),在以下时机运行:

| 时机 | 行为 |
|---|---|
| 模型仓导出前(per [B4 §3](B4-contract-diff.md) 触发) | I3 同时跑 (a) 模型仓 EXPORT vs firmware 模块 contract diff,与 (b) `INS_Out_Bus.yaml` vs firmware `INS_types.h` @ tip 的 mirror diff;**任一**失败 → 导出 hard-fail |
| 每次 mirror PR CI(§4.6.3) | I3 跑 mirror diff 校验 PR 内容;失败 → PR 不可 merge |
| 月度 drift sweep(§4.2.4) | I3 跑 mirror diff;若 drift,bot 自动开补 mirror PR |

#### 4.7.2 HARD-FAIL 语义

drift 被检测时,I3 必须:
- 退出码 non-zero(具体由 [I3](../I-tooling/I3-contract-diff.md) 设计)
- 在 CI 输出明确标识 `INS_Out_Bus mirror drift detected: model schema_version=<N>, firmware tip SHA=<X>, mirror SHA=<Y>`
- 在模型仓 INDEX 决策日志写入 drift 事件(per RULES §10)

**任何模型仓导出**(Plant/FMS/Controller `*.c` 生成)在 INS mirror drift 期间**禁止**通过 — 因为 FMS/Controller 在 firmware 中将消费 firmware 的 `INS_Out_Bus`,而模型仓生成代码内部对 `INS_Out_Bus` 字段的 layout 假设来自 mirror artifact,二者不一致即 silent runtime bug(架构 v1 §17 risk 2 的核心)。

#### 4.7.3 Recover(恢复)

正常恢复路径:bot 自动开 mirror PR(opportunistic 或 monthly sweep)→ reviewer checklist → I3 pass → merge。

#### 4.7.4 Escalate(升级)

| 条件 | 升级动作 |
|---|---|
| mirror PR 开 > 14 天未 merge | **MIL/SIH 仿真冻结**:模型仓侧 simulation runner(由 [I5 sim runner](../I-tooling/I5-sim-runner.md) 落地)拒绝启动新场景(已在跑的不打断);CI 在 PR 描述中标记 BLOCKED;orchestrator 通知 firmware + model 双侧维护者 |
| mirror PR 开 > 28 天 | 升级为 P0 issue,占用主线优先级;主仓 maintainer 介入决定是临时跳过 INS gate(MIL only,SIH/HIL 不可)还是回滚 firmware 改动 |
| firmware tip 变更 > 3 次但 mirror PR 未触发(bot 失效) | monthly sweep 应捕获;若亦失效,manual 触发 + bot 健康检查 issue |

**14 天阈值依据**:Phase 2 多旋翼 MIL 迭代周期通常 ≤ 1 周;14 天给两个完整 review cycle,既不过紧也不让 drift 长期积累。具体阈值由后续工程经验校准,本工作项变更日志记录。

### 4.8 与 F-area 的耦合(下游)

按 [01-design-relationships.md §4.6 F-area](../01-design-relationships.md):

```text
B5 → F1
B5, D1, E1 ⇢ F3
```

本工作项对 F-area 的下游契约:

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [F1 ins_stub 功能设计](../F-ins-contract/F1-ins-stub-functional.md) | → | mirror artifact `INS_Out_Bus.yaml` 字段表 = ins_stub 输出 bus 的**精确 schema**;ins_stub 必须按本镜像产物字段顺序 / 类型 / unit / frame 输出,不得改 / 加 / 减字段;`INS_Status` 和 `INS_Flag` 位号亦遵从镜像([B2 §4.4.10 / §4.4.11](B2-enum-inventory.md) 的 firmware-canonical) |
| [F2 ins_stub 结构设计](../F-ins-contract/F2-ins-stub-structural.md) | → (transitive via F1) | mirror artifact 字段集 = F2 内部块输出 port 的字段集;`Plant_States_Bus → INS_Out_Bus` 字段映射表的右侧字段名以 mirror 为准 |
| [F3 INS_Out_Bus 消费规则](../F-ins-contract/F3-consumption-rules.md) | ⇢ | mirror artifact 锁定的字段集与 `INS_Status` / `INS_Flag` 位号是 F3 落 fallback / validity 处理的字段范围;F3 不可引用 mirror 中不存在的字段 |

约束:F-area 任何对 `INS_Out_Bus` 字段的引用必须只用 mirror artifact 已 lock 的字段。F1/F2/F3 启动时,mirror artifact 必须 ≥ status reviewed(若彼时 firmware commit hash 仍 `<pending>`,F-area 文件标 sibling 引用,与 RULES §6 self-check 第 2 项的 "≥ reviewed 或 co-seal sibling" 规则一致)。

### 4.9 RULES §5 合规 — firmware commit hash 记录

#### 4.9.1 强制记录义务

每个 mirror artifact (`INS_Out_Bus.yaml` + `INS_Out_Bus.firmware-source.md`)**必须**嵌入:

(a) `firmware_commit_sha` — 40-char SHA1,记录在 yaml header block 与 markdown ledger 表
(b) `firmware_path` — firmware 仓内文件相对路径(`src/model/ins/<variant>/lib/INS_types.h` 形式),记录位置同上

#### 4.9.2 CI 拒收策略

CI([I3](../I-tooling/I3-contract-diff.md))在 PR pre-merge 阶段必须校验:
- `firmware_commit_sha` 字段存在且为 40-char hex(正则 `^[0-9a-f]{40}$`)
- `firmware_path` 字段存在,且以 `src/model/ins/` 开头、以 `INS_types.h` 结尾
- 二者**任一**缺失或格式错误 → CI hard-fail,PR 不可 merge

#### 4.9.3 本 B5 文档的 commit hash 占位

本设计文档自身**不**是 mirror artifact(本文档是 process 设计;mirror artifact 是 `INS_Out_Bus.yaml` data 文件)。但本文件 §3 引用 `FMT-Firmware/src/model/ins/<variant>/lib/INS_types.h` 的 commit hash 仍按 [A3](../A-architecture/A3-module-boundaries.md) / [A6](../A-architecture/A6-init-reset-contract.md) / [A7](../A-architecture/A7-time-conventions.md) / [B1](B1-bus-inventory.md) / [B2](B2-enum-inventory.md) / [B3](B3-parameter-schema.md) batch convention 占位 `<pending hash>`,在 INDEX 决策日志登记本工作项时由 orchestrator 同期补齐。FMT-Firmware 仓未在本环境挂载是统一前提。

### 4.10 架构 v1 §17 risk 2 闭合映射

> 架构 v1 §17 risk 2:`INS_Out_Bus` contract drift risk — Because INS now lives entirely in firmware, the model repo only mirrors `INS_Out_Bus`. If firmware changes the bus and the model repo is not updated in lockstep, MIL will silently diverge from real flight behavior and SIH will be the first place the mismatch surfaces. Mitigation: treat `INS_Out_Bus` as a versioned contract; a contract-diff script must compare the model repo's mirrored definition against firmware on every export.

| risk 2 子项 | 本工作项闭合 |
|---|---|
| **Detect**(drift 何时何处暴露) | §4.7.1 — I3 contract diff 在 (a) 模型仓导出前、(b) 每次 mirror PR、(c) monthly sweep 三个时机运行;任一 firmware tip ≠ mirror 立即 hard-fail |
| **Recover**(回到一致) | §4.6 + §4.7.3 — bot-authored mirror PR + reviewer checklist + I3 pass → merge;recovery 路径全自动化(bot)+ 人审 gate(checklist) |
| **Escalate**(系统性卡住时) | §4.7.4 — 14 天未 merge → MIL/SIH 仿真冻结;28 天 → P0 maintainer 介入;bot 失效 → monthly sweep 兜底 + manual + 健康检查 |
| **Versioning**(架构 v1 mitigation 中"versioned contract") | §4.4 — `ins_bus_schema_version` 单调整数 + bump 规则 + G3 manifest 同步 |
| **Source of truth**(架构 v1 mitigation 中"single source of truth") | §4.1.2 — Option A firmware-canonical;§4.1.3 OQ6 闭合 |
| **Diff workflow**(架构 v1 mitigation 中"diff workflow") | §4.6.3 + §4.7 — I3 mirror diff 在三个时机 hard-fail;PR pre-merge 必须 pass |

risk 2 至此**全部子项有对应机制**;具体实现由 [I2](../I-tooling/I2-bus-enum-mirror.md)(bot + 抽取脚本)+ [I3](../I-tooling/I3-contract-diff.md)(diff 脚本)+ [B4](B4-contract-diff.md)(diff 策略 coverage)落地。本工作项是 risk 2 mitigation 的**总流程**载体。

## 5. 已知风险与悬而未决问题

- **firmware commit hash 暂缺**
  - 影响:RULES §5 / §3 表 require 镜像自 firmware 的契约记录 commit hash + 文件相对路径;本文件 §3 仅给路径占位 `FMT-Firmware @ <pending hash>`。
  - 处置:open;评审通过前由 reviewer / orchestrator 在 INDEX 决策日志登记本工作项时补齐(per [A3 §5 第 1 项](../A-architecture/A3-module-boundaries.md) / A6/A7/B1/B2/B3 同样的处置约定)。本工作项 contract_impact=yes 已声明。

- **firmware `<variant>` 子目录结构未确认**
  - 影响:§4.5.2 / §3 表给路径 `FMT-Firmware/src/model/ins/<variant>/lib/INS_types.h`;实际可能是 `src/model/ins/lib/INS_types.h`(单一路径)或多 variant 路径(`multicopter` / `fixwing` 各一份);本环境无法 ground-truth(FMT-Firmware 未挂载)。
  - 处置:open;首次 mirror PR 触发时由 bot + reviewer 实地确认路径,镜像 PR 中 `firmware_path` 字段记录 ground-truth;本文件变更日志同步。若发现多 variant `INS_types.h` 字段不一致,见下条风险。

- **多 variant `INS_types.h` 字段一致性假设**
  - 影响:§4.5.2 假设所有 firmware variant 共享单一 `INS_Out_Bus` schema(模型仓侧只一份 `INS_Out_Bus.yaml`)。若 firmware 后续在不同 variant `INS_types.h` 中定义不同 INS_Out_Bus 字段(估计器输出按机型分化),本设计需扩展为 per-variant mirror artifact。
  - 处置:open;待 firmware ground-truth 确认。若 ground-truth 显示当前 firmware 已分化,本文件 §4.5.1 目录结构需扩为 `model/shared/bus/ins/<variant>/INS_Out_Bus.yaml`,§4.6 PR 工作流需扩为 per-variant PR;变更日志同步。

- **bot 实现 / CI 平台未选型**
  - 影响:§4.2.5 / §4.6.1 / §4.6.4 假设存在 mirror bot(由 I2 落地);具体 bot 运行平台(GitHub Actions / Jenkins / 其他)未选型。
  - 处置:open;由 [I2](../I-tooling/I2-bus-enum-mirror.md) 与模型仓 CI 配置共同决定;本工作项不涉及平台选型。

- **firmware webhook / poll 漏触发概率**
  - 影响:§4.2.2 trigger 依赖 firmware repo webhook 或 cron poll;若 webhook 配置错误或 poll 间隔过长,首次 detect 由 monthly sweep 兜底,期间模型仓导出可能 hard-fail(§4.7.2)— hard-fail 是设计预期,但 reviewer 体感是"无故 fail"。
  - 处置:已规避(monthly sweep + 导出前 I3 diff 双层兜底);具体 webhook 健康检查由 [I2](../I-tooling/I2-bus-enum-mirror.md) + CI 配置落地。

- **firmware 多分支 tracking 未定义**
  - 影响:§4.2.3 默认只跟踪 firmware `master` tip;若 firmware 同时存在 release 分支(例如 `release/2026Q1` 与 `master` 并行),mirror 应追哪条未定。
  - 处置:open;待 firmware 维护策略明确后由本文件后续修订处理;Phase 2 / Phase 3 默认 master only。

- **`INS_Status` enum vs bitfield 形态**(继承自 [B2 §5](B2-enum-inventory.md))
  - 影响:[B2 §4.4.10](B2-enum-inventory.md) 按 enum 形式锁定,[B1 §4.3.1](B1-bus-inventory.md) 按 `uint32` bitfield 锁定;首次 mirror PR 触发时,如果 firmware 实际以 bitfield 实现,B2 走变更日志同步、本文件 mirror artifact 按 firmware 实际形态记录(firmware-canonical 优先)。
  - 处置:open;由首次 mirror PR 决定,变更日志同步。

- **MIL/SIH 冻结的具体执行机制(14 天阈值)**
  - 影响:§4.7.4 给 14 / 28 天阈值与"sim runner 拒绝启动新场景"动作;具体由 [I5 sim runner](../I-tooling/I5-sim-runner.md) 落地的实现细节本工作项不涉及。
  - 处置:open;I5 设计承接;阈值经工程经验校准后本文件变更日志记录。

- **本工作项 contract_impact=yes 的影响范围**
  - 影响:本工作项**直接定义**模型仓侧消费 firmware 契约的镜像流程;变更必须经 INDEX 决策日志登记。
  - 处置:已声明 contract_impact=yes;变更走 RULES §10 + INDEX 登记。本文件不**定义**字段表 / 字节布局(那些在 [B1 §4.3.1](B1-bus-inventory.md) + 本文件 §4.3 锁定的 yaml mirror artifact),仅描述 process。

## 6. 退出条件复核

B5 在 [`00-design-plan.md`](../00-design-plan.md) §4.B 中的退出条件原文:

> **(a) 声明 canonical 来源(firmware 单向 mirror vs 共享 schema 子模块,记录架构 v1 §17 OQ6 的最终选择);(b)** 抽取脚本输入输出、版本标记策略;**(c) 镜像产物必须记录 firmware commit hash + 文件相对路径(RULES §5)**

逐条复核:

| # | 退出条件原文(分解)| 本文档依据 | 状态 |
|---|---|---|---|
| (a) | 声明 canonical 来源(三选项) | §4.1.1 三选项对照表 | 满足 |
| (a) | firmware 单向 mirror vs 共享 schema 子模块 决策 | §4.1.2 选定 Option A;§4.1.2 决策依据 4 条;§4.1.2 权衡 trade-off 3 条 | 满足 |
| (a) | 记录架构 v1 §17 OQ6 的最终选择 | §4.1.3 OQ6 闭合声明:"single source of truth = firmware;schema versioning = `schema_version` 整数;diff workflow = bot PR + I3 hard-fail" | 满足 |
| (b) | 抽取脚本输入 | §4.3.1 输入参数表(`firmware_repo_path` / `firmware_commit_sha` / `firmware_variant` / `output_dir`) | 满足 |
| (b) | 抽取脚本输出 | §4.3.2 双产物 `INS_Out_Bus.yaml` + `INS_Out_Bus.firmware-source.md`;§4.3.3 扩展名选定;§4.3.4 header block 必备字段;§4.3.5 失败语义 | 满足 |
| (b) | 版本标记策略 | §4.4.1 `schema_version` 字段;§4.4.2 bump 规则表;§4.4.3 起点;§4.4.4 与 B3 区分;§4.4.5 G3 manifest 同步 | 满足 |
| (c) | 镜像产物记录 firmware commit hash | §4.3.4 header block 第 1 字段 `firmware_commit_sha`(40-char SHA1,必须);§4.5.3 firmware-source.md 表第 1 行;§4.9.1 强制记录义务;§4.9.2 CI 校验正则 `^[0-9a-f]{40}$` | 满足 |
| (c) | 镜像产物记录文件相对路径 | §4.3.4 header block 第 2 字段 `firmware_path`;§4.5.3 firmware-source.md 表第 2 行;§4.9.1 强制记录义务;§4.9.2 CI 校验前缀 `src/model/ins/...types.h` | 满足 |
| (c) | RULES §5 合规 | §4.9 全节;§4.9.2 CI 拒收策略;§4.9.3 本文件自身 commit hash 占位策略与 A3/A6/A7/B1/B2/B3 batch 一致 | 满足 |

附:对架构 v1 §17 risk 2(INS_Out_Bus contract drift)的 mitigation 闭合见 §4.10(detect / recover / escalate / versioning / source of truth / diff workflow 六项全 mapped)。

退出条件全部满足;状态待 reviewer 升级到 reviewed。

## 7. 下游影响

按 [`01-design-relationships.md §4.2 / §4.6 / §4.9`](../01-design-relationships.md) B5 出边:

```text
B1 → B5
B5 → F1
B5, D1, E1 ⇢ F3
B1, B2, B3 ⇢ I2(B5 与 I2 同 wave 5,B5 提供 I2 的 I/O 契约)
B4 ↔ I3(B5 与 I3 同 wave 5,B5 在 §4.6 / §4.7 引用 I3 作为 merge gate / drift detector)
```

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [F1 ins_stub 功能设计](../F-ins-contract/F1-ins-stub-functional.md) *(未启动,Wave 6)* | → | §4.5 mirror artifact `INS_Out_Bus.yaml` 是 ins_stub 输出 bus 的**精确 schema**(字段顺序 / 类型 / unit / frame / encoding 全部以 mirror 为准);§4.4 `ins_bus_schema_version` 与 ins_stub 实现版本绑定 |
| [F2 ins_stub 结构设计](../F-ins-contract/F2-ins-stub-structural.md) *(未启动,Wave 7)* | → (transitive via F1) | §4.5 mirror artifact 字段集 = F2 内部块输出 port 的字段集;`Plant_States_Bus → INS_Out_Bus` 字段映射的右侧以 mirror artifact 为准 |
| [F3 INS_Out_Bus 消费规则](../F-ins-contract/F3-consumption-rules.md) *(未启动,Wave 9)* | ⇢ | §4.5 mirror artifact 锁定的字段集 + `INS_Status` / `INS_Flag` 位号是 F3 落 fallback / validity 处理的字段范围 |
| [I2 Bus/enum 镜像脚本设计](../I-tooling/I2-bus-enum-mirror.md) *(同 Wave 5,drafting)* | ⇢ | §4.3 抽取脚本 I/O 契约(输入参数表 / 输出文件清单 / header block 必备字段 / 失败语义);§4.6.1 mirror PR 产出物清单是 I2 实现的 bot 行为契约;§4.6.4 是 I2 与 B5 协议 |
| [I3 契约 diff 脚本设计](../I-tooling/I3-contract-diff.md) *(同 Wave 5,drafting)* | ⇢ | §4.6.3 merge gate(I3 必须 pass);§4.7.1 三时机运行(导出前 / mirror PR / monthly sweep);§4.9.2 CI 校验 `firmware_commit_sha` 与 `firmware_path` 格式 |
| [I5 sim runner 脚本设计](../I-tooling/I5-sim-runner.md) *(未启动,Wave 12)* | ⇢ | §4.7.4 14 天 / 28 天升级阈值 → I5 必须实现"拒绝启动新场景"的冻结 hook |
| [G3 日志与可观测性](../G-harness/G3-logging.md) *(未启动,Wave 10)* | ⇢ | §4.4.5 G3 manifest 必须把 `ins_bus_schema_version` + `firmware_commit_sha` + `extracted_at` 入 run metadata |
| [I1 init 脚本设计](../I-tooling/I1-init-script.md) *(未启动,Wave 12)* | ⇢ | §4.5.5 I1 把 `INS_Out_Bus.yaml` 转 `Simulink.Bus` 对象;`INS_Out_Bus` 的 base workspace 字段顺序由 I1 加载逻辑保证与 mirror 一致 |

间接影响(经 F1/F2/F3 以及 INS_Out_Bus 数据流):

- D1 / E1 / D 区其他 / E 区其他在消费 `INS_Out_Bus` 字段时,字段表的真理来源是本工作项 mirror artifact,而非 [B1 §4.3.1](B1-bus-inventory.md) 的消费侧最佳推断(B1 §4.3.1 最佳推断仅用于 Wave 5 之前的下游启动占位;一旦 mirror artifact 落地,以 mirror 为准)。

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-07 | B5 author | 初稿;选定 Option A firmware-canonical 单向 mirror;闭合架构 v1 §17 OQ6 与 risk 2 mitigation;锁定 §4.3 抽取脚本 I/O 契约;commit hash 仍 `<pending hash>` 占位,与 A3/A6/A7/B1/B2/B3 同 batch 模式一致 |
| 2026-05-07 | orchestrator | Reviewer verdict=pass(7/7 准则全部 met);frontmatter 升 reviewed;INDEX 决策日志已登记。非阻塞建议(下一轮迭代):(a) §4.2.4 monthly sweep day 改为确定性锚点(如 first Monday UTC);(b) B3 维护方下次修订时回引 B5 `ins_bus_schema_version` 命名。commit hash 占位 `<pending hash>` 与 A3/B1/B2/B3 同期补齐 |

## Self-check

- [x] frontmatter 完整,字段值合法
- [x] 上游文档全部存在且 status ≥ reviewed,**或**为本工作项所在 co-seal batch 的同批兄弟(架构 v1 已发布;A1 / A3 / A6 / A7 全部 reviewed;B1 / B2 / B3 全部 reviewed 2026-05-07;Wave 5 同期 sibling B4 / I2 / I3 仍 drafting,本工作项**不与**其 co-seal batch 但允许前向引用 — 引用位置已在 §3 / §4.6 / §4.7 标 "drafting in flight" / "by I2 / I3 落地",per RULES §6 self-check 第 2 项的"co-seal sibling"条款语义最佳近似)
- [x] 退出条件逐条复核完成,每条均给出依据(§6 表 9 项 + 架构 v1 §17 risk 2 闭合表)
- [x] 引用路径全部可点击访问(本文件所有引用均使用 RULES §4 规定的相对路径形式;Wave 5 sibling B4 / I2 / I3 + Wave 6+ 下游 F1 / F2 / F3 / G3 / I1 / I5 文件在后续 wave 创建,符合 sibling-or-pending 约定)
- [x] 不存在 RULES §5 禁则中的内容(无 .slx 截图;无可执行 .m 代码;§4.3.4 yaml header block 是 schema 描述非脚本;不复述 firmware 实现细节;不重定义 INS_Out_Bus 字段表 — 字段表的最佳推断在 [B1 §4.3.1](B1-bus-inventory.md),firmware-canonical 真理由 mirror artifact 落地,本文件只描述 process;不重定义 `INS_Status` / `INS_Flag` 数值 — 引用 [B2 §4.4.10 / §4.4.11](B2-enum-inventory.md);不重定义 `INS_PARAM` — [B3 §4.1](B3-parameter-schema.md) 已确认不存在;FMT-Firmware 引用以路径占位 + `<pending hash>` per A3/A6/A7/B1/B2/B3 batch 模式)
- [x] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记(2026-05-07,orchestrator,Wave 5 完成 + B5 contract impact 条目)
- [x] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径(文件相对路径已在 §3 给出 — `FMT-Firmware/src/model/ins/<variant>/lib/INS_types.h`;commit hash 占位 `<pending hash>` 与 A3/A6/A7/B1/B2/B3 同 batch 同期补齐。本工作项是 mirror artifact 自身的设计文档:§4.3.4 / §4.5.3 / §4.9 已强制 mirror artifact 实例嵌入 commit hash + 路径,§4.9.2 CI 拒收非合规 PR — 这是 RULES §5 在落地实例上的强制保证)
- [x] 下游影响已沿关系图识别完毕(§7 已覆盖 [01-design-relationships.md §4.6](../01-design-relationships.md) 全部 B5 出边:`B5 → F1` 与 `B5, D1, E1 ⇢ F3`;并扩展含同 wave 5 sibling I2 / I3 与后 wave I1 / I5 / G3 的承接关系)
- [x] 文档不超出本工作项范围(无越权设计:`INS_Out_Bus` 字段表留 [B1 §4.3.1](B1-bus-inventory.md) + mirror artifact 落地;`INS_Status`/`INS_Flag` 数值留 [B2](B2-enum-inventory.md);`INS_PARAM` 不存在留 [B3](B3-parameter-schema.md);抽取脚本实现留 [I2](../I-tooling/I2-bus-enum-mirror.md);diff 算法留 [B4](B4-contract-diff.md) / [I3](../I-tooling/I3-contract-diff.md);ins_stub 功能 / 结构留 [F1](../F-ins-contract/F1-ins-stub-functional.md) / [F2](../F-ins-contract/F2-ins-stub-structural.md);消费 fallback 留 [F3](../F-ins-contract/F3-consumption-rules.md);harness 拓扑留 [G1](../G-harness/G1-mil-toplevel.md);CI 平台选型留 I2 + 模型仓 CI 配置)
