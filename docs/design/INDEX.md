# FMT-Model-2025b 设计阶段索引

最后更新:2026-05-05
状态:设计阶段进行中。**所有实现工作在设计阶段完成前一律不开始。**

## 总则

- 本仓库当前处于**设计先行**阶段。
- 所有架构/模型/工具的设计文档统一存放于 `docs/design/` 下。
- 设计阶段完成的判定标准:`00-design-plan.md` 中所有工作项达到各自的"退出条件"。
- 实现阶段的工作计划、工作项、闭环开发安排,**待设计阶段结束后**才进入正式排期。

## 顶层规划文件

| 文件 | 用途 |
|---|---|
| [00-design-plan.md](00-design-plan.md) | 设计阶段工作计划:所有设计工作项、产出、退出条件 |
| [01-design-relationships.md](01-design-relationships.md) | 设计项之间的依赖与信息流关系图 |
| [02-execution-playbook.md](02-execution-playbook.md) | 自治执行 playbook:Author/Reviewer/Plan-Auditor 派发模式 |
| [RULES.md](RULES.md) | 设计文档规则:命名、frontmatter、必备章节、禁则、状态机、DoD |
| [_templates/design-doc-template.md](_templates/design-doc-template.md) | 设计文档模板 |
| [INDEX.md](INDEX.md) | 本文件 — 设计阶段索引,随设计文档增长而更新 |

## 配套 Skills(位于 `~/FMT/.claude/skills/`)

| Skill | 用途 |
|---|---|
| `fmt-design-author` | 撰写一个设计工作项的主文档(子代理) |
| `fmt-design-review` | 评审一个设计文档,出具 verdict(子代理) |

## 已完成的上游文档

| 文件 | 说明 |
|---|---|
| [../architecture/2026-05-05-fmt-model-architecture-v1.md](../architecture/2026-05-05-fmt-model-architecture-v1.md) | 架构 v1(已含 INS 划归 firmware 的修订)— 本设计阶段的输入 |

## 设计文档目录结构(占位,随设计推进逐步填充)

```text
docs/design/
├── INDEX.md                          # 本文件
├── 00-design-plan.md                 # 设计阶段工作计划
├── 01-design-relationships.md        # 设计项依赖关系
├── A-architecture/                   # 架构细化(命名、变体、跨速率边界等)
├── B-contracts/                      # bus / enum / parameter 契约设计
├── C-plant/                          # Plant 功能设计 + 结构设计
├── D-fms/                            # FMS 功能设计 + 结构设计 + Stateflow 详细
├── E-controller/                     # Controller 功能设计 + 结构设计 + 算法
├── F-ins-contract/                   # INS_Out_Bus 镜像与 ins_stub 设计
├── G-harness/                        # MIL 顶层、多速率、日志、命令注入
├── H-verification/                   # 验证场景、指标、回归基线
└── I-tooling/                        # 脚本、codegen、契约 diff
```

各子目录在对应设计工作项启动时创建,具体设计文件命名遵循 `<id>-<slug>.md`(例如 `B-contracts/B1-bus-inventory.md`)。

## 已完成设计文档清单

| ID | 标题 | 文件 | 完成日期 | Reviewer Verdict |
|---|---|---|---|---|
| A1 | 架构 v1 评审与缺口闭合 | [A-architecture/A1-v1-review.md](A-architecture/A1-v1-review.md) | 2026-05-05 | pass |
| A2 | 命名与目录约定设计 | [A-architecture/A2-naming-conventions.md](A-architecture/A2-naming-conventions.md) | 2026-05-05 | pass |
| A4 | 跨速率边界设计 | [A-architecture/A4-rate-boundaries.md](A-architecture/A4-rate-boundaries.md) | 2026-05-05 | pass |
| A5 | 变体策略设计 | [A-architecture/A5-variant-strategy.md](A-architecture/A5-variant-strategy.md) | 2026-05-05 | pass |
| A6 | Init/Reset 状态契约 | [A-architecture/A6-init-reset-contract.md](A-architecture/A6-init-reset-contract.md) | 2026-05-05 | pass |
| A7 | 时间与时间戳约定 | [A-architecture/A7-time-conventions.md](A-architecture/A7-time-conventions.md) | 2026-05-05 | pass |
| A8 | 共享库块清单与归属 | [A-architecture/A8-shared-library-roster.md](A-architecture/A8-shared-library-roster.md) | 2026-05-05 | pass |

## 整体计划审查报告

| 日期 | 报告 | 摘要 |
|---|---|---|
| 2026-05-05 | [_audit/2026-05-05-plan-audit.md](_audit/2026-05-05-plan-audit.md) | 42 finding;cross-cutting 缺口(time/clock、init/reset、fault injection、RNG seeding)、E1↔D6 时序冲突、契约 diff 覆盖洞、playbook INDEX-race 等 |

## 决策日志(关键设计决策追溯)

| 日期 | 决策 | 文档锚点 |
|---|---|---|
| 2026-05-05 | INS 划归 FMT-Firmware 原生 C/C++,模型仓不建模 | [架构 v1 §3, §12.3](../architecture/2026-05-05-fmt-model-architecture-v1.md) |
| 2026-05-05 | 采用"设计先行"方法论,完成全面设计后才进入实现 | 本索引 + `00-design-plan.md` |
| 2026-05-05 | RULES §9 评审准则细化:criterion 7 单独不达标可裁决为 pass+non-blocking | RULES §9 |
| 2026-05-05 | RULES §6 / 模板:co-seal batch 兄弟可在 draft 状态互相引用 | RULES §6 |
| 2026-05-05 | RULES §5:镜像自 firmware 的契约必须记录 commit hash + 文件路径 | RULES §5 |
| 2026-05-05 | Playbook §3.1:Author brief 必须显式传 upstream 快照,不通过 INDEX 发现(消除并行 Wave 内 race) | Playbook §3.1 |
| 2026-05-05 | Playbook §3.4:新增 co-seal batch 派发模式(Pattern A/B) | Playbook §3.4 |
| 2026-05-05 | 设计计划由 33 项扩至 37 项:新增 A6 init/reset 状态契约、A7 时间约定、A8 共享库块清单、H4 故障注入目录(基于 [审计 F-02/F-03/F-04 + A1 推荐](_audit/2026-05-05-plan-audit.md)) | 00-design-plan §4.A、§4.H |
| 2026-05-05 | E1 整体推迟到 Wave 8 与 D6 同 batch(原 D6 → E1 强前置与 Wave 顺序矛盾) | 01-design-relationships §6 |
| 2026-05-05 | E5 移至 Wave 10(`E3 → E5` 强前置满足) | 01-design-relationships §6 |
| 2026-05-05 | 多个工作项退出条件加固:契约 diff 覆盖范围(B4 必须含 EXPORT/period/symbol-presence)、参数命名兼容(C3/D5/E4)、§17 OQ1/2/4/5/6 落地到 A5/D1/I5/B5 退出条件、各 I 区脚本要求 worked example、A2/A3 加单位/坐标系约定 | 00-design-plan §4 全部表 |
| 2026-05-05 | 设计协同对扩展:(D3, D4, D6, E1) 合一;新增 (H1, H4) | 01-design-relationships §5 |
| 2026-05-05 | A1 已升至 reviewed(verdict: pass) | A1-v1-review.md frontmatter |
| 2026-05-05 | Wave 2 完成:A2/A4/A5/A6/A7/A8 全部 reviewed(verdict: pass);6 个并行 Author + 6 个并行 Reviewer 派发模式按 playbook §3.1+§3.2 执行 | A-architecture/ + INDEX 已完成清单 |
| 2026-05-05 | A6 (Init/Reset 契约) `contract_impact: yes`:模型仓侧承诺保留 firmware 头文件中 `Plant_init` / `FMS_init` / `Controller_init` 的 `void(void)` 签名;任何 codegen 输出改变 init 签名被视为 hard-fail。Firmware 引用:`FMT-Firmware/src/model/{plant,fms,control}/<vehicle>/lib/{Plant,FMS,Controller}.h` | [A6-init-reset-contract.md §4.2.1](A-architecture/A6-init-reset-contract.md) |
| 2026-05-05 | A7 (时间约定) `contract_impact: yes`:`uint32_t timestamp` 单位 = ms,epoch = zero-from-boot,49.71 天回卷,模块内 dt 走固定步长(非 timestamp 派生),timestamp 仅用于跨模块陈旧检测/日志/profiling。Firmware 引用:`FMT-Firmware/src/model/fms/fms_interface.h:29`、`FMT-Firmware/src/task/vehicle/normal/task_vehicle.c:59,65,81`、`FMT-Firmware/src/module/system/systime.h:99-100` | [A7-time-conventions.md §4.8](A-architecture/A7-time-conventions.md) |
| 2026-05-05 | A7 §5 风险登记:`FMT-Firmware/src/model/fms/template_fms/fms_interface.c:24` 签名 `(void)` 与头文件 `(uint32_t timestamp)` 不一致 — 推荐 firmware 侧修复;模型仓不动 firmware | [A7-time-conventions.md §5, §7](A-architecture/A7-time-conventions.md) |
