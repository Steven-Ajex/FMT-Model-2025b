# FMT-Model-2025b 设计阶段索引

最后更新:2026-05-07
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
| A3 | 模块边界信号清单细化 | [A-architecture/A3-module-boundaries.md](A-architecture/A3-module-boundaries.md) | 2026-05-07 | pass |
| B1 | Bus 清单与 schema 设计 | [B-contracts/B1-bus-inventory.md](B-contracts/B1-bus-inventory.md) | 2026-05-07 | pass |
| B2 | Enum 清单与数值锁定 | [B-contracts/B2-enum-inventory.md](B-contracts/B2-enum-inventory.md) | 2026-05-07 | pass |
| B3 | Parameter schema 设计 | [B-contracts/B3-parameter-schema.md](B-contracts/B3-parameter-schema.md) | 2026-05-07 | pass |
| B4 | 契约 diff 策略设计 | [B-contracts/B4-contract-diff.md](B-contracts/B4-contract-diff.md) | 2026-05-07 | pass |
| B5 | INS_Out_Bus 镜像同步流程 | [B-contracts/B5-ins-bus-mirror.md](B-contracts/B5-ins-bus-mirror.md) | 2026-05-07 | pass |
| I2 | Bus/enum 镜像脚本设计 | [I-tooling/I2-bus-enum-mirror.md](I-tooling/I2-bus-enum-mirror.md) | 2026-05-07 | pass |
| I3 | 契约 diff 脚本设计 | [I-tooling/I3-contract-diff.md](I-tooling/I3-contract-diff.md) | 2026-05-07 | pass |

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
| 2026-05-07 | Wave 3 完成:A3 reviewed(verdict: pass);Author + Reviewer 单工作项串行派发(playbook §3.1+§3.2);A3 闭合 A1 §5.1.4(FMS 输入 mandatory/optional 切分)/§5.1.5(`FMS_Out_Bus` 字段重要性枚举)/§5.1.6(INS 消费侧依赖矩阵 §4.8)/§7.2(FMS passthrough 字段集 §4.6,区分 passthrough-with-shape vs passthrough-direct)/§8.3 + OQ4(`Control_Out_Bus`-as-FMS-input 仅 bus 边界级记录,功能裁决留 D1 §4.10);16 firmware-bus 全覆盖 | [A3-module-boundaries.md](A-architecture/A3-module-boundaries.md) |
| 2026-05-07 | A3 (模块边界信号清单) `contract_impact: yes`:决定 16 个 firmware-mirrored bus 的字段方向(I/O/Optional/Variant-conditional)、Reset 类别(PARAM/HARDCODED/ZERO/PRESERVED,per A6 §4.4)、单位与坐标系(per A2 §4.10)、产出方 nominal rate(per A4 §4.1)。byte 布局留 [B1](B-contracts/B1-bus-inventory.md);A3 标记的 "B1 to confirm" 字段集(motor_cmd 单位、quaternion 顺序、euler 顺序、Mag 单位、wp_array 长度、`FMS_Out_Bus.reset` 字段是否存在、FMS validity echo 字段是否存在)是 B1 显式接受的验证清单。Firmware 引用:`FMT-Firmware/src/model/{plant,fms,control}/<vehicle>/lib/{Plant,FMS,Controller}_types.h` + `FMT-Firmware/src/model/ins/lib/INS_types.h`,commit hash 占位 `<pending hash>`,与 A6/A7 同期补齐 | [A3-module-boundaries.md §3, §4.6, §4.8, §4.10](A-architecture/A3-module-boundaries.md) |
| 2026-05-07 | Wave 4 完成:(B1, B2, B3) 三件套 co-seal batch reviewed(verdict: pass);3 个并行 Author + 1 个共享 Reviewer 派发模式按 playbook §3.4 Pattern B 执行;Reviewer 评审 7×3=21 项 per-doc 准则 + 10 项跨文档一致性(C-1..C-10)全部 met,六条非阻塞建议留待下一轮迭代(B1↔B2 命名同步、B2 增补 IntegratorMethod/FailsafeAction/MixerGeometry/GeofenceShape、INS_Flag 命名前缀显式化、EXPORT 字段集"至少"语义、CONTROL_PARAM.39 归属、att_cmd_quat vs euler 二选一)| B-contracts/ + INDEX 已完成清单 |
| 2026-05-07 | B1 (Bus 清单与 schema) `contract_impact: yes`:锁定 16 个 firmware-mirrored bus 字段级 schema(类型、宽度、cumulative offset 占位、单位、坐标系、encoding/quantization、validity 依赖、passthrough/variant/cmd_mask 标签);`FMS_Out_Bus` 33 字段 ledger 闭合 A1 §5.1.5;Vec3/Quat/Euler/INS_Status/INS_Flag/Waypoint 子 schema 单一定义;little-endian ARM Cortex-M 假设;cumulative offset 标 `(B4 verify alignment)`,首跑 B4/I3 修订后晋级 locked。Firmware 引用同 A3。enum 数值留 [B2](B-contracts/B2-enum-inventory.md);PARAM 字段留 [B3](B-contracts/B3-parameter-schema.md);cmd_mask 位语义留 D6/E1(Wave 8 co-seal) | [B1-bus-inventory.md §3, §4.7, §4.9, §4.10](B-contracts/B1-bus-inventory.md) |
| 2026-05-07 | B2 (Enum 清单与数值锁定) `contract_impact: yes`:14 contract-locked enums(VehicleStatus/VehicleState/VehicleExtState/PilotMode/FlightMode/CtrlMode/FailsafeState/MissionState/INS_Status/INS_Flag bit 位/GPS_FixType/ErrorCode/GCS_CmdType/CmdMaskBit 位)+ 2 placeholder(GearState/LandingState)+ 1 internal-fault boundary。Drift policy P-1..P-6(immutable values / end-of-list-only / lockstep / deprecate-by-mark / underlying-width fail / firmware-canonical default)+ Diff coverage D-1..D-8(per-name exact / missing=fail / firmware-only=warn / width=fail);`cmd_mask` bit 位号 owned,bit 语义留 D6/E1。numeric values 标 `(B4 verify)`,首跑 B4/I3 后 §10 changelog 升 locked。Firmware 引用同 B1 | [B2-enum-inventory.md §3, §4.2, §4.3, §4.4](B-contracts/B2-enum-inventory.md) |
| 2026-05-07 | B3 (Parameter schema) `contract_impact: yes`:三 PARAM struct 共 116 字段(PLANT_PARAM=34 / FMS_PARAM=43 / CONTROL_PARAM=39),逐字段 R-1..R-5 五规则首匹配 runtime-tunable / compile-inlined 分类(96 R-T = 83% / 20 C-I = 17%),各带一句话 rationale(闭合 audit F-08 / arch v1 §14.3)。多旋翼默认值 PX4-class 占位标 `(B4 verify)`,leaf 兼容性框架 §4.9 三模板待 C3/D5/E4 填(闭合 audit F-33)。EXPORT struct + model_info[] 6-field ledger(model_name/vehicle_class/schema_version/firmware_compat_hash/build_timestamp/param_field_count);`period` 写入留 I4。bus 字段留 B1;enum 数值留 B2。Firmware 引用同 B1 | [B3-parameter-schema.md §3, §4.2.2, §4.6.4, §4.9](B-contracts/B3-parameter-schema.md) |
| 2026-05-07 | Wave 5 完成:(B4, I3) co-seal pair + B5 + I2 全部 reviewed=pass。派发模式:B4+I3 Pattern A 单 Author + 共享 Reviewer(per-doc 7×2=14 + cross-doc X-1..X-8 全部 met),B5/I2 各 1 Author + 1 Reviewer(7/7 met)。Wave 5 闭合 arch v1 §17 OQ6(`INS_Out_Bus` schema canonical=Option A firmware-canonical 单向 mirror)+ risk 2(契约 drift detect/recover/escalate 三段映射)+ audit F-31/F-32(B4 (a)..(f) 6 类 24 个 Check ID)+ audit F-18(I3 / I2 各含 worked example)。共 13 条非阻塞建议留待下一轮迭代 | B-contracts/ + I-tooling/ + INDEX 已完成清单 |
| 2026-05-07 | B4 (契约 diff 策略) `contract_impact: yes`:24 个 Check ID 跨 6 个 clause(a01..a05 bus 字段顺序+Simulink→C 类型+累计字节宽 / b01..b04 enum 名+数值精确 / c01..c04 *_PARAM 字段顺序+类型+storage class+总字节宽 / d01..d04 *_EXPORT 字段顺序+`period` 数值+`model_info[]` 长度+总字节宽 / e01..e03 entry 符号 *_init / *_step / f01..f03 PARAM 全局符号);严重级 22 fail / 2 warn(b03 firmware-only end-of-list / c04 PARAM byte 估计);零容忍策略;4 阶段触发(pre-export / post-export / pre-merge gate / pre-release gate);`--allow-drift` 受限(reason+expiry+INDEX entry 必填,main/release 与 (e)/(f) 禁用);JSON+Markdown 报告 schema。Firmware 引用同 B1 | [B4-contract-diff.md §4.1, §4.2, §4.3, §4.4, §4.5](B-contracts/B4-contract-diff.md) |
| 2026-05-07 | B5 (INS_Out_Bus 镜像同步流程) `contract_impact: yes`:Option A firmware-canonical 单向 mirror(arch v1 §17 OQ6 closure);.yaml 文本 mirror 主形式(git-diff 友好);`ins_bus_schema_version` 整数计数器(单调,never-reuse,与 B3 EXPORT semver `schema_version` 不冲撞);Bot-PR + 8 项 reviewer checklist + I3 hard-fail merge gate;3-tier drift escalation(14d MIL/SIH freeze → 28d P0 → 月度 sweep)。Mirror artifact 落地实例强制 `firmware_commit_sha` + `firmware_path` header block,CI regex `^[0-9a-f]{40}$` 拒收非合规 PR(RULES §5 落地)。Firmware 引用:`FMT-Firmware/src/model/ins/<variant>/lib/INS_types.h`,commit hash 占位 `<pending hash>` 与 A3/A6/A7/B1/B2/B3/B4 同期补齐 | [B5-ins-bus-mirror.md §4.1, §4.3, §4.4, §4.6, §4.7, §4.9, §4.10](B-contracts/B5-ins-bus-mirror.md) |
| 2026-05-07 | I2 + I3 (Wave 5 实现侧) `contract_impact: no`:I2 Python 3.11+ + pycparser + pyyaml + jsonschema,dual-format 输出(Simulink Bus + YAML mirror,JSON Schema 等价校验),三层人工审查闸口 + 7 项 reviewer checklist,§4.9 worked example 含 B5 §4.3.4 header block。I3 24 Check ID 与 B4 §4.1 一一对应,Python 3.10+ + pycparser + pyelftools(libclang fallback),ELF 主 + .c 静态 fallback;`contract-diff/{post-export,pre-merge,pre-release}` CI status-check 名;exit 0/1/2/3 与 B4 §4.4.1 一致;§4.7 worked example 演示 (a)/(b) 双 drift 类。I2/I3 自身不定义契约;契约由 B1/B2/B3 拥有,保护策略由 B4 拥有,跨仓库 PR 流程由 B5 拥有。无独立 INDEX contract impact 条目 | [I2-bus-enum-mirror.md](I-tooling/I2-bus-enum-mirror.md) + [I3-contract-diff.md](I-tooling/I3-contract-diff.md) |
| 2026-05-05 | A6 (Init/Reset 契约) `contract_impact: yes`:模型仓侧承诺保留 firmware 头文件中 `Plant_init` / `FMS_init` / `Controller_init` 的 `void(void)` 签名;任何 codegen 输出改变 init 签名被视为 hard-fail。Firmware 引用:`FMT-Firmware/src/model/{plant,fms,control}/<vehicle>/lib/{Plant,FMS,Controller}.h` | [A6-init-reset-contract.md §4.2.1](A-architecture/A6-init-reset-contract.md) |
| 2026-05-05 | A7 (时间约定) `contract_impact: yes`:`uint32_t timestamp` 单位 = ms,epoch = zero-from-boot,49.71 天回卷,模块内 dt 走固定步长(非 timestamp 派生),timestamp 仅用于跨模块陈旧检测/日志/profiling。Firmware 引用:`FMT-Firmware/src/model/fms/fms_interface.h:29`、`FMT-Firmware/src/task/vehicle/normal/task_vehicle.c:59,65,81`、`FMT-Firmware/src/module/system/systime.h:99-100` | [A7-time-conventions.md §4.8](A-architecture/A7-time-conventions.md) |
| 2026-05-05 | A7 §5 风险登记:`FMT-Firmware/src/model/fms/template_fms/fms_interface.c:24` 签名 `(void)` 与头文件 `(uint32_t timestamp)` 不一致 — 推荐 firmware 侧修复;模型仓不动 firmware | [A7-time-conventions.md §5, §7](A-architecture/A7-time-conventions.md) |
