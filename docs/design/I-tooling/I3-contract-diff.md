---
work_item: I3
title: 契约 diff 脚本设计
upstream: ["架构 v1", "A1", "A2", "A3", "A6", "B1", "B2", "B3", "B4"]
contract_impact: no
status: reviewed
authored_at: 2026-05-07
last_reviewed_at: 2026-05-07
reviewer_verdict: pass
---

# I3 契约 diff 脚本设计

## 1. 目的

为 [B4 契约 diff 策略设计](../B-contracts/B4-contract-diff.md) 定义的 24 行覆盖矩阵给出**脚本侧实现设计**:管线、CLI、输入解析、每条 Check ID 的实现算法、报告生成、CI 集成、worked example。本文件是 HOW(实现),与 co-seal 兄弟 B4(WHAT,策略)成对。本文件**不**重定义任何契约或 diff 阈值 — 全部引用 B4。本文件**只设计脚本**(per RULES §5,设计阶段不写可执行代码;脚本细节以伪代码描述,标 `pseudo`)。

## 2. 范围

**在范围:**

- **§4.1 脚本架构**:管线图 + 模块切分(parser / canonical model / differ / reporter)
- **§4.2 CLI**:入参 / 入参组合 / 退出码 / `--help` 例
- **§4.3 输入解析**:firmware `*_types.h` 解析、模型仓 B1/B2/B3 markdown(或 I2 YAML 镜像)解析、codegen 产物 / ELF 符号表解析
- **§4.4 Per-check 实现**:对应 B4 §4.1 矩阵 24 行逐项实现规约(B4-a01..a05 / b01..b04 / c01..c04 / d01..d04 / e01..e03 / f01..f03 全覆盖)
- **§4.5 报告输出**:JSON 与 Markdown 生成(与 B4 §4.5 schema 完全一致)
- **§4.6 CI 集成**:env vars / 运行时预算 / artifact 上传 / status-check name
- **§4.7 Worked example**:模拟一种带 (a) 添加 + (b) `VehicleMode` 数值漂移的输入,展示 JSON / Markdown / 退出码
- **§4.8 实现注**:语言选型 / 第三方库 / firmware-side 编译依赖
- **§4.9 失败模式与限制**:脚本检测不到的情形(alignment / 宏类型别名 / 跨 vehicle 漂移 / 静态分析死角)

**不在范围(由其他工作项处理):**

- diff 策略 / coverage 矩阵 / 阈值 / 触发时机 — 由 [B4 契约 diff 策略设计](../B-contracts/B4-contract-diff.md) 处理;本文件只**实现** B4
- bus / enum / PARAM 字段定义 — 由 [B1](../B-contracts/B1-bus-inventory.md) / [B2](../B-contracts/B2-enum-inventory.md) / [B3](../B-contracts/B3-parameter-schema.md) 处理
- bus / enum YAML 镜像格式 — 由 [I2 Bus/enum 镜像脚本设计](I2-bus-enum-mirror.md) 处理(I3 可选消费 I2 输出;若 I2 未提供 YAML,I3 直接解析 B1/B2/B3 markdown)
- ert.tlc codegen 配置 — 由 [I4 Codegen 配置脚本设计](I4-codegen-config.md) 处理(I3 在 codegen 产物上做 post-export check,但不规定 codegen 配置项)
- 仿真 / 回归脚本流水线 — 由 [I5 仿真运行/批量回归脚本设计](I5-sim-runner.md) 处理(I5 在 export 流程中调用 I3 作为子步骤)
- INS_Out_Bus 镜像源决策(单向 / 共享子模块)— 由 [B5 INS_Out_Bus 镜像同步流程](../B-contracts/B5-ins-bus-mirror.md) 处理;I3 把 `INS_Out_Bus` 视为 B1 §4.3.1 一条 bus,统一按 B4-a* 规则 diff
- 实际 firmware tip 数值 — FMT-Firmware 仓未挂载;本文件占位 `FMT-Firmware @ <pending hash>`

## 3. 依赖

| 上游 | 引用位置 | 用途 |
|---|---|---|
| 架构 v1 §15 lockstep / §17 Risk 1, Risk 6 | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | 契约 diff 在 export 链上的位置 |
| [A1 架构 v1 评审与缺口闭合](../A-architecture/A1-v1-review.md) (status: reviewed, 2026-05-05) | A1 §5.1.7 / §5.1.9 critical | I3 是 critical 闭合的实现侧 |
| [A2 命名与目录约定设计](../A-architecture/A2-naming-conventions.md) (status: reviewed, 2026-05-05) | A2 §4.8 R-8.1 顶层模型根名 | 入口符号 / PARAM 全局符号名匹配规则 |
| [A3 模块边界信号清单细化](../A-architecture/A3-module-boundaries.md) (status: reviewed, 2026-05-07) | A3 §4.3..§4.9 字段元数据 | bus 字段集合的设计意图来源(由 B1 落到字节级)|
| [A6 Init/Reset 状态契约](../A-architecture/A6-init-reset-contract.md) (status: reviewed, 2026-05-05) | A6 §4.2 init 行为契约 | 入口符号契约(`*_init` / `*_step`)|
| [B1 Bus 清单与 schema 设计](../B-contracts/B1-bus-inventory.md) (status: reviewed, 2026-05-07) | B1 §4.1 类型映射 / §4.2..§4.8 16 bus 字段表 / §4.6 子 schema | (a) 检查的模型仓侧 baseline |
| [B2 Enum 清单与数值锁定](../B-contracts/B2-enum-inventory.md) (status: reviewed, 2026-05-07) | B2 §4.2 drift policy / §4.4.1..§4.4.16 enum 值表 | (b) 检查的模型仓侧 baseline + 末尾追加判定规则 |
| [B3 Parameter schema 设计](../B-contracts/B3-parameter-schema.md) (status: reviewed, 2026-05-07) | B3 §4.3 / §4.4 / §4.5 PARAM 字段表 / §4.6.4 storage class budget / §4.7 EXPORT struct + model_info[] ledger / §4.8 diff coverage 要求 | (c) (d) 检查的模型仓侧 baseline |
| **co-seal batch (B4, I3)**:[B4 契约 diff 策略设计](../B-contracts/B4-contract-diff.md) (sibling, draft) | per [01-design-relationships.md §5 协同对 3](../01-design-relationships.md) — entry 3 = `(B4, I3)` | **本工作项与 B4 同 batch(Wave 5 协同定稿)。B4 owns 策略;I3 owns 实现。本文件 §4.4 子节与 B4 §4.6 cross-ref 一一对应;允许引用 B4 draft per RULES §6 self-check 第 2 项(co-seal 同批互引)** |
| [I2 Bus/enum 镜像脚本设计](I2-bus-enum-mirror.md) (status: not started; planned Wave 5) | (可选输入) | 若 I2 产出 YAML 镜像,I3 优先消费 YAML;否则解析 B1/B2/B3 markdown |
| firmware `*_types.h` + codegen 产物(diff 的另一端) | `FMT-Firmware/src/model/{plant,fms,control}/<vehicle>/lib/{Plant,FMS,Controller}_types.h`;`FMT-Firmware/src/model/ins/lib/INS_types.h`;以及含 `*_PARAM` / `*_EXPORT` 全局声明的相关头文件;**FMT-Firmware @ \<pending hash\>**(commit hash 由 INDEX 决策日志维护方在登记本工作项时补齐;FMT-Firmware 仓未挂载在本环境)| firmware 端真值;脚本读取 firmware `*_types.h` 作为 text |

注:**co-seal batch 名 = (B4, I3)**(per [01-design-relationships.md §5 协同对 3](../01-design-relationships.md) entry 3)。本工作项 contract_impact=`no` — I3 自身仅是脚本设计,不直接定义 firmware-visible 契约;契约定义在 B1/B2/B3,契约保护策略在 B4。但 I3 的实现错误会导致 B4 策略失效,因此评审等同 contract-adjacent。

环境注:FMT-Firmware 仓未挂载;本文件以 firmware path + `<pending hash>` 占位描述脚本输入,不指定具体 firmware tip 数值。worked example(§4.7)使用合成数据演示。

## 4. 设计内容

### 4.1 脚本架构

#### 4.1.1 管线图

```text
                     ┌────────────────────────────────────────────────────┐
                     │  Inputs                                            │
                     │                                                    │
   firmware repo ───►│  (1) firmware *_types.h  (text)                    │
                     │  (2) firmware EXPORT impl (.c with PLANT_EXPORT=…) │
                     │  (3) codegen artifacts (.c/.h + optional ELF)      │
   model repo   ───►│  (4) B1/B2/B3 markdown  OR  I2 YAML mirror          │
                     └────────────────────────────────────────────────────┘
                                        │
                                        ▼
        ┌──────────────────────────────────────────────────────────┐
        │  Stage 1 — Parsers                                       │
        │  - C header parser  (*_types.h → AST: structs/enums)     │
        │  - C source scanner (*.c → globals + function defs)      │
        │  - ELF symbol-table reader (optional, if ELF available)  │
        │  - Model-side parser (B1/B2/B3 .md  OR  I2 YAML)         │
        └──────────────────────────────────────────────────────────┘
                                        │
                                        ▼
        ┌──────────────────────────────────────────────────────────┐
        │  Stage 2 — Canonical model                               │
        │  In-memory representation of:                            │
        │    - bus_schema[bus_name]    = [(field, type, width)]    │
        │    - enum_schema[enum_name]  = [(member, value)]         │
        │    - param_schema[param]     = [(field, type, class)]    │
        │    - export_schema[export]   = [(field, type), period,   │
        │                                  model_info_len]         │
        │    - symbols                  = set of names             │
        │  Built once for model side, once for firmware side.      │
        └──────────────────────────────────────────────────────────┘
                                        │
                                        ▼
        ┌──────────────────────────────────────────────────────────┐
        │  Stage 3 — Differ                                        │
        │  For each Check ID in B4 §4.1:                           │
        │    apply matcher (B4-a01..f03)                           │
        │    emit coverage_row(id, status, expected, actual,       │
        │                      diff_text, source_*)                │
        │  Apply severity → status decision (B4 §4.2)              │
        └──────────────────────────────────────────────────────────┘
                                        │
                                        ▼
        ┌──────────────────────────────────────────────────────────┐
        │  Stage 4 — Reporter                                      │
        │  - emit report.json   (B4 §4.5.1 schema)                 │
        │  - emit report.md     (B4 §4.5.2 schema)                 │
        │  - decide overall_status + exit_code (B4 §4.4.1)         │
        │  - return                                                │
        └──────────────────────────────────────────────────────────┘
                                        │
                                        ▼
                     <output-dir>/contract-diff/<run-ts>/{report.json, report.md}
```

#### 4.1.2 模块切分(脚本内部架构)

| 模块 | 职责 | 输入 | 输出 |
|---|---|---|---|
| `cli` | 解析命令行;校验入参组合 | `argv` | parsed args |
| `parser.c_header` | 解析 firmware `*_types.h`(struct / enum / typedef) | path → text | AST nodes |
| `parser.c_source` | 扫描 `.c` 全局变量与函数定义 | path → text | symbols |
| `parser.elf` | 读 ELF symbol table | path → ELF | symbol set |
| `parser.model` | 读 B1/B2/B3 markdown 或 I2 YAML | path → text | model schema |
| `model.canonical` | 构建 in-memory canonical | parser outputs | canonical schemas |
| `differ.bus` | 实现 B4-a01..a05 | model + firmware bus schemas | coverage rows |
| `differ.enum` | 实现 B4-b01..b04 | model + firmware enum schemas | coverage rows |
| `differ.param` | 实现 B4-c01..c04 | model + firmware PARAM schemas + codegen | coverage rows |
| `differ.export` | 实现 B4-d01..d04 | model + firmware EXPORT schemas | coverage rows |
| `differ.symbols` | 实现 B4-e01..e03, f01..f03 | symbols set + acceptance criteria | coverage rows |
| `reporter.json` | 输出 `report.json` | coverage rows + run metadata | file |
| `reporter.md` | 输出 `report.md` | 同上 | file |
| `gate` | 决定 overall status + exit code(per B4 §4.4.1)| coverage rows | 退出码 |

### 4.2 CLI

#### 4.2.1 入参定义

| 参数 | 类型 | 必填 | 默认值 | 含义 |
|---|---|---|---|---|
| `--firmware-repo-path` | path | yes | — | FMT-Firmware 仓本地路径 |
| `--firmware-sha` | string (40-char) | yes | — | firmware 仓应锁定到的 commit SHA;脚本运行前 verify(`git -C <path> rev-parse HEAD == sha`)|
| `--model-repo-path` | path | yes | — | FMT-Model-2025b 本地路径 |
| `--model-yaml-mirror` | path | no | (auto-detect) | I2 YAML 镜像目录;若给定优先于 markdown 解析 |
| `--codegen-output-path` | path | conditional | — | codegen 产物目录(post-export / pre-merge / pre-release 触发时必填;pre-export 可省)|
| `--vehicle` | enum (`multicopter`/`fixwing`/`vtol`/`boat`/`car`/`*`) | no | `multicopter` | per [A5](../A-architecture/A5-variant-strategy.md) Phase 2 = multicopter |
| `--trigger` | enum (`pre-export`/`post-export`/`pre-merge`/`pre-release`) | yes | — | 决定 §4.3 (B4) 矩阵参与子集 |
| `--output-dir` | path | no | `./reports/contract-diff/` | report.json / report.md 输出根目录 |
| `--severity-threshold` | enum (`fail`/`warn`) | no | `fail` | 退出码升级阈值;`fail` = 默认(只 fail 退非零);`warn` = 严格(warn 也退非零,用于 nightly 强保护)|
| `--allow-drift` | flag (boolean) | no | `false` | 触发 B4 §4.4.5 例外通道;必须搭配 `--allow-drift-reason` + `--allow-drift-expires` |
| `--allow-drift-reason` | string | conditional | — | `--allow-drift` 必填;无 reason 时 I3 拒绝执行 |
| `--allow-drift-expires` | date (YYYY-MM-DD) | conditional | — | `--allow-drift` 必填;过期则 `--allow-drift` 自动失效 |
| `--quiet` / `--verbose` | flag | no | — | 调整 stdout 详细程度;不影响 report 内容 |
| `--help` | flag | no | — | 打印用法 |

入参组合校验(脚本启动时):

1. `--firmware-sha` 必须等于 `git -C <firmware-repo-path> rev-parse HEAD`;不一致 → exit 3 (tool error)。
2. `--trigger=pre-export` → 允许 `--codegen-output-path` 缺;其他 trigger 必填。
3. `--allow-drift` → 必填 reason 与 expires;两者任一缺失 → exit 3。
4. `--trigger=pre-merge` 或 `--trigger=pre-release` 且检测到分支为 main / release/* → 忽略 `--allow-drift`(per B4 §4.4.5 规则 5)。
5. `--vehicle=*` 时,脚本对每个已注册 vehicle 跑一次,合并报告。

#### 4.2.2 退出码

(锚定 [B4 §4.4.1](../B-contracts/B4-contract-diff.md))

| 退出码 | 含义 |
|---|---|
| 0 | pass — 无 fail 无 warn 无 error |
| 1 | warn-only — 至少一行 status=warn,无 fail / error |
| 2 | hard-fail — 至少一行 status=fail,与 warn / error 共存可能 |
| 3 | tool error — 入参校验失败 / parse 异常 / firmware path 缺失 / I/O 失败 |

注:`--severity-threshold=warn` 时,warn-only 升级到退出码 1(默认即如此);若进一步要求 warn 阻塞 export,export 工具(I5)按退出码 1 自行决定阻塞,本脚本不二次升级。

#### 4.2.3 `--help` 例

```text
$ contract_diff --help
contract_diff — FMT-Model-2025b contract diff tool (implements B4)

Usage:
  contract_diff --firmware-repo-path <path>    \
                --firmware-sha <sha40>          \
                --model-repo-path <path>        \
                --trigger <pre-export|post-export|pre-merge|pre-release>  \
                [--codegen-output-path <path>]  \
                [--vehicle multicopter]         \
                [--output-dir ./reports/contract-diff/] \
                [--severity-threshold fail|warn] \
                [--allow-drift --allow-drift-reason "..." --allow-drift-expires YYYY-MM-DD]

Examples:
  # post-export check on multicopter
  contract_diff --firmware-repo-path /opt/FMT-Firmware \
                --firmware-sha abcdef1234... \
                --model-repo-path . \
                --codegen-output-path ./build/codegen \
                --trigger post-export \
                --vehicle multicopter

  # bootstrap with allow-drift
  contract_diff ... --trigger post-export \
                --allow-drift \
                --allow-drift-reason "bootstrap baseline" \
                --allow-drift-expires 2026-06-01

Exit codes:
  0  pass
  1  warn-only
  2  hard-fail
  3  tool error
```

### 4.3 输入解析

#### 4.3.1 firmware 头文件解析(模块 `parser.c_header`)

输入:firmware `*_types.h` 文本。

输出 AST:`{structs: [...], enums: [...], typedefs: [...]}`,每个节点含 `name, fields/members, source_path, source_line`。

伪代码:

```pseudo
function parse_c_header(path):
    text = read_file(path)
    text = preprocess_for_diff(text)        # strip comments; expand simple #defines; do NOT compile
    ast  = c_grammar.parse(text)            # struct / enum / typedef
    return collect_nodes(ast)
```

实现注:不需要完整 C compiler;只需 typedef / struct / enum 子集。推荐 `pycparser` 或同等纯 Python 解析器(per §4.8)。`#include` 不递归 — 只解析显式给定的头文件路径。

边界案例:

- `__attribute__((packed))` → 在节点上打 `packed=true` 标记;影响 B4-a03 / d04 的字节宽度计算(packed → 不加 padding)。
- 嵌套 struct typedef(B1 §4.6 子 schema)→ 单独节点存放,主 struct 字段类型引用之。
- enum 隐式底层类型(无 `: uint8_t`)→ 节点 `underlying_type=None`,B4-b04 检查时按 firmware ABI 推断(C99: `int`,但 ARM EABI 通常按取值范围紧致,需结合 `sizeof(<enum>)` 实测;静态推断给 best-effort,标 `(verify by ELF)`)。
- `#define` 数组长度 → 简单展开;复杂表达式跳过并标 `parse-skip` 再走 ELF 路径回填。

#### 4.3.2 firmware C 源扫描(模块 `parser.c_source`)

输入:含 `*_PARAM` / `*_EXPORT` 全局声明的 `.c` 文件(或 codegen 产物 `.c`)。

输出:`{globals: [(name, type, init_value)], function_defs: [name]}`。

伪代码:

```pseudo
function scan_c_source(path):
    text = read_file(path)
    globals      = regex_find_globals(text)     # match e.g. "PLANT_PARAM_TYPE PLANT_PARAM = {...};"
    function_defs = regex_find_function_defs(text)
    return globals, function_defs
```

边界案例:

- 函数定义 vs 声明:用"花括号紧跟"启发(`int foo(...) {`)区分;声明加分号无花括号忽略。
- 全局变量初值含嵌套 brace:计 brace depth 直到平衡;捕获完整初值文本作为后续 EXPORT `period` 数值提取的输入。

#### 4.3.3 ELF symbol table 读取(模块 `parser.elf`)

输入:codegen 产物 ELF 路径(可选)。

输出:`{defined_symbols, undefined_symbols}`,each = set of strings。

伪代码:

```pseudo
function read_elf_symbols(path):
    if not exists(path): return None        # ELF optional
    elf = open_elf(path)
    return (set of defined, set of undefined)
```

实现注:用纯 Python `pyelftools`(per §4.8)。若 ELF 不可用,符号检查回退到 `parser.c_source` 的 `function_defs` + `globals` 集。

#### 4.3.4 模型仓侧解析(模块 `parser.model`)

两条路径:

1. **YAML mirror 路径(优先)**:若 [I2](I2-bus-enum-mirror.md) 已生成 `<model-repo>/build/contract-mirror/{bus,enum,param,export}.yaml`,直接读取(I2 owns YAML schema)。
2. **Markdown 解析路径(回退)**:解析 [B1](../B-contracts/B1-bus-inventory.md) §4.2..§4.8 / [B2](../B-contracts/B2-enum-inventory.md) §4.4.x / [B3](../B-contracts/B3-parameter-schema.md) §4.3 / §4.4 / §4.5 / §4.7 各表。

Markdown 表解析伪代码:

```pseudo
function parse_b1_markdown(path):
    text = read_file(path)
    sections  = split_by_heading(text, "###" or "####")
    for each bus_section in sections matching "###? 4\\.\\d.*Bus":
        rows = parse_markdown_table(bus_section)
        bus_schema[bus_name] = [(row.field, row.type, int(row.width), int(row.offset)) for row in rows]
    return bus_schema
```

边界案例:

- Markdown 中 `(B4 to verify)` 标注 → 标 `verify=true`,但 baseline 仍按 Type / Width 列录入。
- B1 §4.7 ledger 多个子表(a)/(b)/(c)/(d)/(e)合并为 `FMS_Out_Bus` 单一字段表(按 ledger Order 排列)。
- B3 storage class 列 `R-T` / `C-I` 直接录入 `param_schema[].class`。

### 4.4 Per-check 实现

每子节对应 [B4 §4.1](../B-contracts/B4-contract-diff.md) 中相应 Check ID 集;实现规约 + diff 算法 + 报告行格式。

#### 4.4.a Bus 检查(B4-a01..a05)

**B4-a01 bus 字段顺序**

```pseudo
for each bus in B1_16_buses:
    expected = model.bus_schema[bus]   # list of field names in order
    actual   = firmware.bus_schema[bus]
    if expected.field_names != actual.field_names (order-sensitive):
        emit fail row with diff_text = unified_diff(expected, actual)
    else:
        emit pass row
```

**B4-a02 类型映射**

For each (model_field, firmware_field) at same index: map model_field.simulink_type via [B1 §4.1.2](../B-contracts/B1-bus-inventory.md) table → expected_c_type. Compare with firmware_field.c_type byte-exactly. Differences (including array length, nested typedef name) → fail.

**B4-a03 累计字节宽度**

```pseudo
for each bus:
    expected_size = sum_with_natural_alignment(model.bus_schema[bus])
    actual_size   = compute_struct_sizeof(firmware.bus_schema[bus], packed=node.packed)
    if expected_size != actual_size:
        emit fail row with diff_text = "expected sizeof=%d, firmware sizeof=%d, delta=%d" %
```

`compute_struct_sizeof` 根据 packed / natural alignment 选择算法。注:firmware sizeof 可经"静态计算"(I3 自带,基于 AST + ABI 规则 ARM EABI)或经 ELF 直接取(若有);两路一致则 OK,不一致则报 firmware-side ambiguity → exit 3 error。

**B4-a04 endianness**

```pseudo
expected = "little-endian"   # B1 §4.1.1
actual   = parse_target_endianness(firmware_repo_path)
            # via ELF e_ident or target triple in build config
if actual != expected:
    emit fail
```

**B4-a05 嵌套子 schema**

按 B1 §4.6 子 schema 列表(`quat[4]`, `pos_ned[3]`, `vel_ned[3]`, `ang_rate_b[3]`, `euler[3]`, `INS_Status`, `INS_Flag`, `Waypoint`)逐项以 a01+a02+a03 算法 diff 子 typedef。

**报告行格式**(各 a* 检查统一):

```text
{ id: "B4-a01", clause: "a", name: "<bus> field order",
  status: "pass|fail",
  expected: "[f1, f2, f3, ...]",
  actual:   "[f1, f2, f3', ...]",
  diff_text: "  f1\n  f2\n- f3\n+ f3'\n",
  source_model:    "docs/design/B-contracts/B1-bus-inventory.md#§4.<n>",
  source_firmware: "FMT-Firmware/.../<bus>_types.h:<line>" }
```

#### 4.4.b Enum 检查(B4-b01..b04)

**B4-b01 enum 类型存在性**

```pseudo
for each enum_name in B2_enum_list:
    if enum_name not in firmware.enum_schema:
        emit fail row "enum type missing in firmware"
    else:
        emit pass
```

**B4-b02 成员名+数值精确**

```pseudo
for each enum in B2 ∩ firmware:
    expected_members = set of (name, value) from B2
    actual_members   = set of (name, value) from firmware
    common = expected_members ∩ actual_members
    if (name, value) mismatch within shared names:
        emit fail row with detail
```

**B4-b03 双向覆盖(末尾追加判定)**

```pseudo
for each enum:
    model_only    = expected_members - actual_members  (by name)
    firmware_only = actual_members - expected_members  (by name)

    if model_only is non-empty:
        emit fail row "model-only addition (forbidden by B2 §4.2 P-3)"
    if firmware_only is non-empty:
        if all (firmware_only_member.value > max(expected_members.value))
           and (firmware_only_members are contiguous starting at max+1):
            emit warn row "firmware-only end-of-list addition (lockstep pending sync)"
        else:
            emit fail row "firmware-only addition NOT at end (contract-breaking)"
```

注意 deprecate hole 边界:若 firmware 弃用某成员(B2 §4.2 P-4 "保留名+原数值"),firmware 端此成员仍存在,model 端亦保留,故 P-4 行为不会经 B4-b03 出 fail/warn — only B4-b02 验证一致。

**B4-b04 底层整型宽度**

```pseudo
for each enum:
    expected_width = B2 enum.underlying_type.width    # default 1 (uint8); bitfield 4
    actual_width   = sizeof(firmware enum)             # via static AST or ELF
    if mismatch: emit fail row "underlying width drift, P-5 contract breach"
```

#### 4.4.c PARAM 检查(B4-c01..c04)

**B4-c01 PARAM 字段顺序**

类似 B4-a01,but 三个 PARAM struct: `PLANT_PARAM_TYPE` / `FMS_PARAM_TYPE` / `CONTROL_PARAM_TYPE`。

**B4-c02 PARAM 字段类型**

类似 B4-a02。

**B4-c03 PARAM 字段 storage class 一致性**

```pseudo
for each PARAM_struct:
    for each field in B3.param_schema[PARAM_struct]:
        expected_class = field.class                 # 'R-T' or 'C-I'
        actual_class   = derive_from_codegen(field.name, codegen_output)
        if mismatch: emit fail
```

`derive_from_codegen`: 在 codegen `.c` 中查 `extern volatile <type> <name>`(R-T) 或 inline 常量(C-I)。

**B4-c04 PARAM 总字节宽度**(severity=`warn`)

```pseudo
expected_estimate = B3 §4.10.1 estimate
actual_size       = sizeof(<MODULE>_PARAM_TYPE)  via static AST or ELF
if actual_size != expected_estimate:
    emit warn row "estimate vs firmware sizeof differ; B3 §4.10.1 estimate excludes padding"
```

#### 4.4.d EXPORT 检查(B4-d01..d04)

**B4-d01 EXPORT 字段顺序**

类似 B4-c01 over `PLANT_EXPORT_TYPE` / `FMS_EXPORT_TYPE` / `CONTROL_EXPORT_TYPE`。

**B4-d02 `period` 数值**

```pseudo
for each EXPORT_struct in [PLANT_EXPORT, FMS_EXPORT, CONTROL_EXPORT]:
    expected_period = B3 §4.7.<n>.default       # Plant=1, FMS=20, Controller=5 (per arch v1 §13)
    actual_period   = parse_export_init_value(firmware EXPORT instance .c, field='period')
    if expected_period != actual_period: emit fail
```

`parse_export_init_value` 从 `parser.c_source` 输出的 globals 中查 `<MODULE>_EXPORT = { .period = <val>, ... }`。

**B4-d03 `model_info[]` 长度**

```pseudo
expected_N = parse from B3 §4.7.<n>          # B3 declared N or '<N B4 verify>'
actual_N   = parse from firmware typedef     # char model_info[N]
if expected_N != actual_N: emit fail (or 'verified' if expected was placeholder, 回填 B3)
```

注:若 B3 §4.7 N 是占位 `<N B4 verify>`,首次 run 把 firmware N 视作 expected,emit `pass-with-note` 并回写 INDEX 决策日志(prompting B3 update);之后 run 按实际 N 比对。

**B4-d04 EXPORT 总字节宽度**

类似 B4-a03 over EXPORT structs。

#### 4.4.e 入口符号检查(B4-e01..e03)

```pseudo
for each (module, init_sym, step_sym) in [
    ('Plant',      'Plant_init',      'Plant_step'),
    ('FMS',        'FMS_init',        'FMS_step'),
    ('Controller', 'Controller_init', 'Controller_step')]:
    found = (init_sym in elf.defined_symbols) AND (step_sym in elf.defined_symbols)
    if not found:
        # ELF unavailable fallback: scan .c function defs
        found = (init_sym in c_source.function_defs) AND (step_sym in c_source.function_defs)
    if not found:
        emit fail row "entry symbols missing for <module>"
    else:
        emit pass
```

注:若 ELF 不可用且 `.c` 不可读 → emit `error` (per B4 §4.5.1 silent-skip 守则)。

#### 4.4.f PARAM 全局符号检查(B4-f01..f03)

```pseudo
for each (module, global_name) in [
    ('Plant',      'PLANT_PARAM'),
    ('FMS',        'FMS_PARAM'),
    ('Controller', 'CONTROL_PARAM')]:
    found = global_name in elf.defined_symbols
    if not found:
        # fallback: scan .c globals
        found = any(g.name == global_name for g in c_source.globals)
    if not found:
        emit fail row "PARAM global symbol missing for <module>"
    else:
        emit pass
```

### 4.5 报告输出

#### 4.5.1 JSON 输出

格式与 [B4 §4.5.1](../B-contracts/B4-contract-diff.md) 完全一致。I3 实现细节:

```pseudo
function write_report_json(coverage_rows, run_meta, output_path):
    obj = {
        "tool_version":     I3_TOOL_VERSION,
        "firmware_commit":  run_meta.firmware_sha,
        "model_commit":     get_git_head(model_repo_path),
        "run_at":           now_iso8601_utc(),
        "trigger":          run_meta.trigger,
        "vehicle":          run_meta.vehicle,
        "overall_status":   compute_overall(coverage_rows),
        "exit_code":        compute_exit_code(coverage_rows),
        "allow_drift":      run_meta.allow_drift_block,
        "coverage":         [row.to_dict() for row in coverage_rows],
        "summary":          summarize(coverage_rows)
    }
    write_json(output_path, obj, sort_keys=False, indent=2)
```

`compute_overall` / `compute_exit_code` 严格按 [B4 §4.4.1](../B-contracts/B4-contract-diff.md) 4 行表实现。

#### 4.5.2 Markdown 输出

格式与 [B4 §4.5.2](../B-contracts/B4-contract-diff.md) 完全一致。I3 在同一份 in-memory `coverage_rows` 上渲染:

```pseudo
function write_report_md(coverage_rows, run_meta, output_path):
    fail_rows  = [r for r in coverage_rows if r.status == 'fail']
    warn_rows  = [r for r in coverage_rows if r.status == 'warn']
    error_rows = [r for r in coverage_rows if r.status == 'error']
    pass_rows  = [r for r in coverage_rows if r.status == 'pass']

    sections = [
        render_header(run_meta),
        render_summary(coverage_rows),
        render_failures(fail_rows),
        render_warnings(warn_rows),
        render_errors(error_rows),
        render_pass_collapsed(pass_rows)
    ]
    write_text(output_path, "\n\n".join(sections))
```

注:JSON 与 Markdown 必同源(同一 in-memory `coverage_rows`),保证两份报告内容一致;由 §4.6 CI 强制断言。

### 4.6 CI 集成

#### 4.6.1 必需环境变量

| Env var | 用途 |
|---|---|
| `FMT_FIRMWARE_REPO_PATH` | firmware 仓本地缓存路径(由 CI runner 提前 fetch)|
| `FMT_FIRMWARE_SHA` | firmware tip(对 pre-merge gate)/ release SHA(对 pre-release gate)|
| `FMT_MODEL_REPO_PATH` | 默认 `${GITHUB_WORKSPACE}` 或等价 |
| `FMT_VEHICLE` | `multicopter`(Phase 2 默认)|
| `FMT_OUTPUT_DIR` | 默认 `${GITHUB_WORKSPACE}/reports/contract-diff/` |

#### 4.6.2 运行时预算

| Trigger | 预期运行时长 | budget cap |
|---|---|---|
| pre-export | < 5 s(轻量,parse + bus/enum/param/export schema diff,no codegen artifacts)| 30 s |
| post-export | 30 s ~ 2 min(parse + diff + 符号扫描 + ELF 读)| 5 min |
| pre-merge | 同 post-export + firmware fetch overhead | 10 min |
| pre-release | 同 pre-merge + 多 vehicle 循环(若启用)| 15 min |

#### 4.6.3 Artifact 上传路径

每次 CI run:

- `${FMT_OUTPUT_DIR}/contract-diff/<run-timestamp>/report.json`
- `${FMT_OUTPUT_DIR}/contract-diff/<run-timestamp>/report.md`

CI 配置上传上述目录为 build artifact,保留 ≥ 90 天供 triage。pre-release run 的 artifact 长期保留(挂在 release tag)。

#### 4.6.4 Status-check 名

(锚定 [B4 §4.8 D-8](../B-contracts/B4-contract-diff.md))

| Trigger | Status-check name |
|---|---|
| `post-export` | `contract-diff/post-export` |
| `pre-merge` | `contract-diff/pre-merge` |
| `pre-release` | `contract-diff/pre-release` |
| (pre-export 在 CI 上一般不单独跑 status-check;由 pre-merge gate 包含)| (n/a) |

CI 在分支保护规则中要求 `contract-diff/pre-merge` = success-or-warn 才能合入 main。

#### 4.6.5 退出码到 CI 状态映射

| 退出码 | CI status |
|---|---|
| 0 | success |
| 1 | success-with-warnings(per B4 §4.4.3)|
| 2 | failure(阻塞 PR / release)|
| 3 | error(阻塞;tool 故障)|

#### 4.6.6 双源 firmware fetch

CI runner 应:

1. checkout firmware 仓到 `$FMT_FIRMWARE_REPO_PATH`;`git checkout $FMT_FIRMWARE_SHA`。
2. 启动 contract_diff;脚本自身会 verify `git rev-parse HEAD == $FMT_FIRMWARE_SHA`(防错位 fetch)。
3. 失败立即 exit 3。

### 4.7 Worked example(per audit F-18 / F-32 / B4 audit clauses + 任务 brief 退出条件)

#### 4.7.1 模拟漂移输入

模拟 firmware tip 与模型仓 schema 之间的两处 drift:

1. **(a) bus 字段添加** — firmware `FMS_Out_Bus` 在 `error_code` 之后追加新字段 `new_advisory_field` (`uint8`),模型仓 [B1 §4.7](../B-contracts/B1-bus-inventory.md) 未声明此字段。
2. **(b) enum 数值漂移** — firmware `VehicleMode` 把 `MODE_HOLD` 数值从 5 改为 7(B2 §4.4.5 锁定为 5)。

调用:

```text
$ contract_diff \
    --firmware-repo-path /opt/FMT-Firmware \
    --firmware-sha abcdef1234567890abcdef1234567890abcdef12 \
    --model-repo-path /home/user/FMT-Model-2025b \
    --codegen-output-path ./build/codegen/multicopter \
    --trigger post-export \
    --vehicle multicopter \
    --output-dir ./reports/contract-diff/

Exit code: 2
```

#### 4.7.2 期望 JSON 输出(report.json)

```text
{
  "tool_version": "0.1.0",
  "firmware_commit": "abcdef1234567890abcdef1234567890abcdef12",
  "model_commit":    "1234567890abcdef1234567890abcdef12345678",
  "run_at": "2026-05-08T03:14:25Z",
  "trigger": "post-export",
  "vehicle": "multicopter",
  "overall_status": "fail",
  "exit_code": 2,
  "allow_drift": {"active": false, "reason": null, "expires": null},
  "coverage": [
    {
      "id": "B4-a01",
      "clause": "a",
      "name": "FMS_Out_Bus field order",
      "severity": "fail",
      "status": "fail",
      "expected": "[..., 'home_lat_deg', 'home_lon_deg', 'home_alt_m', 'error_code', 'failsafe_state', 'timestamp']",
      "actual":   "[..., 'home_lat_deg', 'home_lon_deg', 'home_alt_m', 'error_code', 'new_advisory_field', 'failsafe_state', 'timestamp']",
      "diff_text": "  error_code\n+ new_advisory_field\n  failsafe_state\n  timestamp\n",
      "source_model":    "docs/design/B-contracts/B1-bus-inventory.md#§4.7",
      "source_firmware": "FMT-Firmware/src/model/fms/multicopter/lib/FMS_types.h:208"
    },
    {
      "id": "B4-a03",
      "clause": "a",
      "name": "FMS_Out_Bus cumulative byte width",
      "severity": "fail",
      "status": "fail",
      "expected": "sizeof(FMS_Out_Bus) = 312",
      "actual":   "sizeof(FMS_Out_Bus) = 316",
      "diff_text": "delta = +4 bytes (likely natural-alignment padding around new uint8 field)",
      "source_model":    "docs/design/B-contracts/B1-bus-inventory.md#§4.7 'Total bus byte width'",
      "source_firmware": "FMT-Firmware/src/model/fms/multicopter/lib/FMS_types.h (sizeof)"
    },
    {
      "id": "B4-b02",
      "clause": "b",
      "name": "VehicleMode member numeric values",
      "severity": "fail",
      "status": "fail",
      "expected": "MODE_HOLD = 5  (B2 §4.4.5)",
      "actual":   "MODE_HOLD = 7",
      "diff_text": "- MODE_HOLD = 5\n+ MODE_HOLD = 7\n  (P-1 immutable values violation; B2 §4.2)",
      "source_model":    "docs/design/B-contracts/B2-enum-inventory.md#§4.4.5",
      "source_firmware": "FMT-Firmware/src/model/fms/multicopter/lib/FMS_types.h:142"
    },
    {
      "id": "B4-b03",
      "clause": "b",
      "name": "VehicleMode bidirectional coverage",
      "severity": "fail",
      "status": "pass",
      "expected": "model and firmware members are equal sets by name",
      "actual":   "model and firmware members are equal sets by name",
      "diff_text": "",
      "source_model":    "docs/design/B-contracts/B2-enum-inventory.md#§4.4.5",
      "source_firmware": "FMT-Firmware/src/model/fms/multicopter/lib/FMS_types.h:142"
    },
    {
      "id": "B4-e01",
      "clause": "e",
      "name": "Plant entry symbols",
      "severity": "fail",
      "status": "pass",
      "expected": "{Plant_init, Plant_step} ⊆ ELF symbols",
      "actual":   "{Plant_init, Plant_step, Plant_initialize} ⊆ ELF symbols",
      "diff_text": "",
      "source_model":    "(codegen artifact)",
      "source_firmware": "(N/A — checked on model-side artifact)"
    },
    {
      "id": "B4-f01",
      "clause": "f",
      "name": "PLANT_PARAM global symbol",
      "severity": "fail",
      "status": "pass",
      "expected": "PLANT_PARAM ∈ ELF defined symbols",
      "actual":   "PLANT_PARAM ∈ ELF defined symbols",
      "diff_text": "",
      "source_model":    "(codegen artifact)",
      "source_firmware": "(N/A — checked on model-side artifact)"
    }
    /* ... 其他 18 行 status=pass 省略,完整 24 行在真实 report 中全列 ... */
  ],
  "summary": {"total": 24, "pass": 21, "warn": 0, "fail": 3, "error": 0}
}
```

注:`B4-b03` 在本例 status=pass,因为 firmware 与 model 在 `VehicleMode` 上**成员名集合等价**(只是 `MODE_HOLD` 数值不同);B4-b03 双向覆盖判定基于"name 集合",所以未触发 b03 fail;b02(数值精确)捕获到了漂移。这正是把 b02 / b03 拆开的设计动机。

#### 4.7.3 期望 Markdown 输出(report.md 摘要)

```text
# Contract Diff Report

- **Tool version:** 0.1.0
- **Firmware commit:** abcdef1234567890abcdef1234567890abcdef12 (path: FMT-Firmware/src/model/...)
- **Model commit:** 1234567890abcdef1234567890abcdef12345678
- **Run at:** 2026-05-08T03:14:25Z
- **Trigger:** post-export
- **Vehicle:** multicopter
- **Overall status:** **FAIL**
- **Exit code:** 2
- **Allow-drift:** inactive

## Summary

| Total | Pass | Warn | Fail | Error |
|---:|---:|---:|---:|---:|
| 24 | 21 | 0 | 3 | 0 |

## Failures (severity=fail, status=fail)

| Check ID | Clause | Name | Expected | Actual | Diff (excerpt) |
|---|---|---|---|---|---|
| B4-a01 | a | FMS_Out_Bus field order | [..., error_code, failsafe_state, timestamp] | [..., error_code, new_advisory_field, failsafe_state, timestamp] | + new_advisory_field |
| B4-a03 | a | FMS_Out_Bus cumulative byte width | sizeof = 312 | sizeof = 316 | delta = +4 bytes |
| B4-b02 | b | VehicleMode member numeric values | MODE_HOLD = 5 | MODE_HOLD = 7 | - 5 / + 7  (P-1 violation) |

## Warnings (severity=warn or warn-eligible)

(none)

## Errors (tool-side)

(none)

## Pass list (collapsed)

21 checks passed: B4-a02, B4-a04, B4-a05, B4-b01, B4-b03, B4-b04, B4-c01, B4-c02, B4-c03, B4-c04, B4-d01, B4-d02, B4-d03, B4-d04, B4-e01, B4-e02, B4-e03, B4-f01, B4-f02, B4-f03 (and additional pass rows from sub-bus checks).
```

#### 4.7.4 Fail / warn / pass 行的辨识

| 行 | severity | status | 备注 |
|---|---|---|---|
| B4-a01 (FMS_Out_Bus 字段添加)| fail | **fail** | 主漂移 1;触发 post-export abort |
| B4-a03 (FMS_Out_Bus 字节宽度)| fail | **fail** | a01 的副作用(B1 §4.7 总宽度声明被破坏)|
| B4-b02 (VehicleMode 数值)| fail | **fail** | 主漂移 2;触发 P-1 不可变值违反 |
| B4-b03 (VehicleMode 双向覆盖)| fail | pass | 数值漂移不破坏 name 集合等价,b03 不触发 |
| B4-e01..e03 / f01..f03 | fail | pass | 符号存在性 OK |
| 其余 18 行 | mostly fail (1 warn=B4-c04, B4-b03 has both) | pass | 无漂移 |
| **Overall** | — | **FAIL** | 至少一行 status=fail → exit code 2 |

#### 4.7.5 退出码

```text
$ echo $?
2
```

`exit_code=2` 触发(per B4 §4.4.1)export abort;codegen 产物不写入 `export/firmware/<module>/`;PR / release 阻塞。

### 4.8 实现注

| 项 | 选择 | 依据 |
|---|---|---|
| 主要语言 | **Python 3.10+**(parser / differ / reporter)| 纯文本解析 / JSON 生成 / pyelftools / pycparser 生态成熟;无需运行 Simulink |
| 次要语言 | **MATLAB**(仅 Bus Object introspection,如需要)| 若 Simulink Bus Object 镜像需直接读 SLDD,MATLAB headless `slreportgen` 可用;但本设计主路径 = 静态文本(B1/B2/B3 markdown 或 I2 YAML),不需要 MATLAB |
| Python 第三方库 | `pycparser`(C header parser)+ `pyelftools`(ELF symbol table)+ `pyyaml`(若 I2 提供 YAML)+ `mistletoe` 或 `markdown-it-py`(markdown 表解析,fallback) | 全部 pure Python,无原生编译依赖;CI runner 只需 `pip install pycparser pyelftools pyyaml mistletoe` |
| firmware 端编译依赖 | **无**(头文件按 text 读)| `pycparser` 不需要 firmware 编译产物;只需 `*_types.h` 文件存在 |
| firmware 路径解析 | git checkout SHA → 静态读 `*_types.h` | 不调用 firmware build system;不需要 toolchain |
| codegen 产物读取 | `.c` / `.h` 文本 + 可选 ELF | ELF 缺失时回退 `.c` 静态扫描(per §4.4.e/f fallback)|
| 性能 | 单次 run < 5 min(post-export);`pycparser` 解析 < 1 s/file × ~10 files ≈ 10 s | 满足 §4.6.2 budget |
| 跨平台 | Linux / macOS / Windows(WSL)| pure Python;CI runner 任意 |
| 单元测试 | I3 自身需有 unit test(在 [tests/contract/](../../../tests/) 子目录)| 用 fixture firmware header 验证每条 Check ID 的 fail / warn / pass 路径 |

### 4.9 失败模式与限制

下列情形脚本**无法**检测;在文档中显式声明:

1. **结构 alignment / packing 隐藏在相同字段顺序后**:firmware 对某 struct 加 `__attribute__((packed))` 而模型仓未声明;若字段顺序与类型 byte-exact 等价,B4-a01/a02 pass,但实际 sizeof 不同 — B4-a03 (cumulative byte width) 应捕获,但若两侧都 packed 或都 natural-aligned,padding 数值匹配则隐藏。Mitigation:CI 上同时跑"静态 sizeof 计算"与"firmware sizeof"两路对比(§4.4.a B4-a03)。
2. **宏驱动的类型别名**(`#define UINT16_T uint16_t` 等)在 pycparser 默认配置下可能漏解析;复杂宏需用 `pycparser.preprocess_file` + 真实 cpp。**Mitigation**:I3 实现层提供 `--preprocess-with-cpp` 选项;否则跳过的宏在报告中标 `parse-skip`。
3. **跨 vehicle leaf 漂移**:模型仓 B1/B2/B3 是 vehicle-agnostic,但 firmware `<vehicle>` 路径下可能含 vehicle-specific 字段。Phase 2 仅 multicopter,不构成问题;Phase 3+ 多 vehicle 时,I3 入参 `--vehicle=*` 将逐个跑,但跨 vehicle 之间 schema 一致性不在脚本范围。
4. **firmware build 时常量计算**:firmware `*_types.h` 中 `#define ARRAY_LEN (N_MOT_MAX * 3)` 类宏 ARRAY 长度依赖 build-time `N_MOT_MAX`;脚本静态展开有限。**Mitigation**:I3 优先从 ELF 读取 sizeof / 数组长度;`.c` fallback 时报 `parse-skip` 显式登记。
5. **enum 隐式底层类型**(无 `: uint8_t`)在不同编译器下宽度不同(C99 `int` 但 ARM EABI 紧致)。**Mitigation**:B4-b04 在静态推断+ELF 实测两路;不一致 → exit 3 error 不出 false pass。
6. **codegen 产物中入口符号被 strip**:嵌入式 firmware build pipeline 可能 strip ELF 符号表(release build);此时 §4.4.e/f 自动 fallback 到 `.c` 扫描;若 `.c` 也不可用(只剩二进制)→ emit `error`(per silent-skip 守则)。
7. **runtime parameter linkage drift**:R-T 字段在 GCS 在线下发参数时,firmware 实例化的 PARAM 数值可能与 codegen 产物的 default 不同;但本检查只比 schema(字段顺序+类型),不比运行时值。运行时值漂移由 GCS / firmware 团队管理,不属契约 diff 范围。
8. **`#ifdef` 切换的字段集**:firmware 头文件可能有 `#ifdef CONFIG_X` 包围的可选字段;`pycparser` 默认按 `#define` 状态展开。**Mitigation**:I3 入参 `--firmware-config-defines KEY=VAL` 显式传 firmware build config;否则按 firmware 默认 build config 展开。
9. **deprecate hole 与 jump 数值**:per B2 §4.2 P-2 / P-4,deprecate 不删成员但保留原数值;新成员追加在最大值之后。若 firmware 跳号(reserve range)或在 deprecate hole 中插入新值,B4-b03 末尾追加判定可能误判。**Mitigation**:遇到不连续追加 → 降级到 `fail`(safe default,见 B4 §5 风险)。

## 5. 已知风险与悬而未决问题

- **firmware commit hash 暂缺**
  - 影响:[RULES §5 / §3](../RULES.md) require 引用 firmware 路径 + commit hash;本文件 §3 给路径占位 `FMT-Firmware @ <pending hash>`(per B1/B2/B3/B4 同样占位)。
  - 处置:open;评审通过前由 reviewer / orchestrator 在 INDEX 决策日志登记本工作项时补齐。

- **`pycparser` 与现代 C 头文件兼容性**
  - 影响:firmware `*_types.h` 若用 C11 / C17 特性(`_Static_assert`, `_Alignas`)或 GNU 扩展(`typeof`),`pycparser` 默认不识别。
  - 处置:I3 实现期 evaluate `pycparser` 真实头文件兼容性;若不足,切换到 `clang.cindex`(libclang Python 绑定)— 更强但有原生依赖(libclang.so)。本设计文档保留 pycparser 为首选,libclang 为 fallback。

- **ELF 不可用时的 silent-skip 风险**
  - 影响:若 codegen pipeline 不产 ELF,§4.4.e/f 退化到 `.c` 扫描;`.c` 静态扫描无法 100% 等价 ELF symbol table(weak symbols / linker-resolved 未必现)。
  - 处置:I3 在 ELF 可用时优先读 ELF;不可用时 `.c` fallback 并在 report 中标 `verify_method=c_scan`(让 reviewer 知道 fallback 已激活)。

- **CI 上 firmware 仓 fetch 时长**
  - 影响:每次 pre-merge gate 都 fetch firmware tip,网络慢时超 budget。
  - 处置:CI runner 配置 git shallow clone + cache;fetch 在 contract_diff 之前的 stage 执行,不计入 §4.6.2 budget。

- **YAML mirror vs markdown 解析的双源风险**
  - 影响:I3 优先消费 [I2](I2-bus-enum-mirror.md) YAML mirror;若 I2 与 B1/B2/B3 markdown 不同步(I2 滞后),I3 报告将基于过期 baseline 出错误 pass。
  - 处置:I3 启动时验证 YAML mirror commit_hash == B1/B2/B3 markdown commit_hash(by I2 metadata);不匹配 → exit 3 error。或者:`--no-yaml-mirror` 强制 markdown 解析。

- **多 worked example 维护负担**
  - 影响:§4.7 worked example 内嵌 firmware path / SHA 占位;首次真实 run 时需更新此节示例数值。
  - 处置:本文件 §4.7 标"假数据";真实首跑后由 author 走 §8 变更日志更新一次,作为 reference fixture。

- **`--allow-drift` 滥用与审计**
  - 影响:per B4 §4.4.5,`--allow-drift` 必填 reason+expires;但脚本无法防止开发者编造 reason。
  - 处置:I3 在 report.json 中嵌入 `allow_drift.reason` 与 `expires`;CI artifact 长期保留供审计回查;月度聚合监控滥用次数。本风险设计层声明,实施由 I5 / CI 配置承担。

- **本工作项 contract_impact=no,但实质 contract-adjacent**
  - 影响:I3 不直接定义契约,但 I3 实现的错误 / 缺失 = B4 策略失效 = 契约保护落空。
  - 处置:RULES §6 self-check 第 5 项要求 contract_impact=yes 才登记 INDEX;本工作项 no,故无须登记;但作者在 §3 注与 §6 复核中显式声明此 contract-adjacent 性质,由 reviewer 在 evaluating I3 退出条件时按高保护标准评审(per [01-design-relationships.md §5 协同对 3](../01-design-relationships.md))。

- **co-seal batch 协同风险**
  - 影响:B4 兄弟 draft 在协同评审中可能修订 §4.5 schema 或 §4.1 矩阵 ID 命名;本文件需随之更新。
  - 处置:per RULES §6 self-check 第 2 项 + 01-design-relationships.md §5 entry 3,co-seal batch (B4, I3) Wave 5 协同定稿;评审时 reviewer 检查策略 / 实现两侧字段对齐(I3 §4.4.x 子节 ↔ B4 §4.1 ID;I3 §4.5 schema ↔ B4 §4.5 schema)。

## 6. 退出条件复核

I3 在 [`00-design-plan.md §4.I`](../00-design-plan.md) 中的退出条件原文:

> 与 B4 协同(co-seal):输入/输出格式、字段级 diff 报告、CI 接入方式;**实现 B4 全部覆盖项(bus/enum/PARAM/EXPORT/period/model_info + 符号存在性);至少一个 worked example(模拟一个差异输入,展示报告输出)**

逐条复核:

| # | 退出条件分解 | 本文档依据 | 状态 |
|---|---|---|---|
| 1 | 与 B4 协同(co-seal)| §3 batch 声明;§4.6 cross-ref to B4 §4.1 矩阵;同 batch 互引 per RULES §6 第 2 项 | 满足 |
| 2 | 输入格式 | §4.2 CLI 入参表;§4.3 三类输入(firmware header / model-side B1/B2/B3 or I2 YAML / codegen 产物)解析子节 | 满足 |
| 3 | 输出格式 | §4.5.1 JSON(锚定 B4 §4.5.1);§4.5.2 Markdown(锚定 B4 §4.5.2);双源同 in-memory 渲染保证一致 | 满足 |
| 4 | 字段级 diff 报告 | §4.5.1 `coverage[]` 字段含 `id` / `clause` / `expected` / `actual` / `diff_text` / `source_model` / `source_firmware`;§4.4 各 Check ID 的报告行格式 | 满足 |
| 5 | CI 接入方式 | §4.6 子节(env vars / runtime budget / artifact upload / status-check name / 退出码 → CI status 映射 / firmware fetch)| 满足 |
| 6 | (a) bus 字段顺序 + 类型 + 字节相等 | §4.4.a B4-a01/a02/a03/a04/a05 五子节;每子节给 diff 算法 + 报告行 | 满足 |
| 7 | (b) enum 数值精确相等 | §4.4.b B4-b01/b02/b03/b04 四子节;含末尾追加判定与 deprecate hole 边界 | 满足 |
| 8 | (c) `*_PARAM` 字段顺序 + 类型 | §4.4.c B4-c01/c02/c03/c04 四子节;含 storage class + 总字节宽度 | 满足 |
| 9 | (d) `*_EXPORT` 字段顺序 + `period` + `model_info[]` 长度 | §4.4.d B4-d01/d02/d03/d04 四子节 | 满足 |
| 10 | 入口符号存在性(`*_init` / `*_step` × 3 模块)| §4.4.e B4-e01/e02/e03 三检查 + ELF + `.c` fallback 算法 | 满足 |
| 11 | PARAM 全局符号存在性(`*_PARAM` × 3 模块,任务 brief audit 追加 (f))| §4.4.f B4-f01/f02/f03 三检查 | 满足 |
| 12 | 至少一个 worked example | §4.7 完整 worked example(模拟 (a) `FMS_Out_Bus.new_advisory_field` 添加 + (b) `VehicleMode.MODE_HOLD` 数值 5→7 漂移)— 含调用、JSON 输出、Markdown 输出、退出码、行辨识表 | 满足 |

附:对任务 brief 显式追加的 audit 要求:

| 任务 brief 要求 | 本文档依据 | 状态 |
|---|---|---|
| 实现 B4 全部覆盖项 | §4.4.a/b/c/d/e/f 六子节,逐一对应 B4 §4.1 24 行(B4-a01..f03)| 满足 |
| Worked example 含 (a) JSON / (b) Markdown / (c) fail vs warn 辨识 / (d) 退出码 | §4.7.2 / §4.7.3 / §4.7.4 / §4.7.5 四子子节 | 满足 |
| Worked example 用 B1 §4.7 / B2 §4.4 真实字段名 | §4.7 用 `FMS_Out_Bus.error_code` / `failsafe_state` / `VehicleMode.MODE_HOLD` 等 B1 / B2 真实字段名 | 满足 |

附:与 B4 §4.1 矩阵的逐行覆盖证明:

| B4 Check ID | I3 实现章节 | 状态 |
|---|---|---|
| B4-a01..a05 | §4.4.a | 全覆盖 |
| B4-b01..b04 | §4.4.b | 全覆盖 |
| B4-c01..c04 | §4.4.c | 全覆盖 |
| B4-d01..d04 | §4.4.d | 全覆盖 |
| B4-e01..e03 | §4.4.e | 全覆盖 |
| B4-f01..f03 | §4.4.f | 全覆盖 |

24 / 24 全覆盖。

退出条件全部满足;状态:`draft`,待评审升级到 `reviewed`。

## 7. 下游影响

按 [`01-design-relationships.md §4.9 / §5`](../01-design-relationships.md) 本工作项的出边:

```text
B4 ↔ I3                              (co-seal batch entry 3)
(间接)I3 ⇢ I5(sim runner 在 export 流程中调用 I3)
(间接)I3 ⇢ G3 / I4(I3 消费 codegen 产物;I4 codegen 配置必须保证 I3 检查面)
```

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [B4 契约 diff 策略设计](../B-contracts/B4-contract-diff.md) *(co-seal sibling, draft)* | ↔ | §4.4 各 Check ID 的实现规约 → 验证 B4 §4.1 矩阵全部可实现;§4.5 报告生成 → 兑现 B4 §4.5 schema;§4.6 CI 集成 → 兑现 B4 §4.7 lockstep |
| [I2 Bus/enum 镜像脚本设计](I2-bus-enum-mirror.md) *(未启动,Wave 5)* | ⇢ (可选下游) | I3 §4.3.4 声明可消费 I2 YAML mirror;若 I2 实现 YAML 输出,I3 自动适配;若 I2 仅 markdown,I3 解析 markdown |
| [I4 Codegen 配置脚本设计](I4-codegen-config.md) *(未启动,Wave 12)* | ⇢ | I3 §4.4.e/f 要求 codegen 产物含入口符号 / PARAM 全局符号 → I4 codegen 配置必须保证;I3 §4.4.c B4-c03 要求 storage class 与 B3 §4.6.4 一致 → I4 配置必须按 B3 设置 `Tunable=on/off` |
| [I5 仿真运行/批量回归脚本设计](I5-sim-runner.md) *(未启动,Wave 12)* | ⇢ | I3 §4.2 CLI 与 §4.6.5 退出码 → I5 在 export pipeline 中调用 I3 的接口;I5 错误处理基于 I3 退出码 |
| [B5 INS_Out_Bus 镜像同步流程](../B-contracts/B5-ins-bus-mirror.md) *(未启动,Wave 5)* | (无直接出边)| I3 把 `INS_Out_Bus` 视作 B1 §4.3.1 一条 bus,不与 B5 镜像方向交互;若 B5 决定 canonical 来源是 firmware 单向 mirror,I3 §4.4.a 的"firmware 端"自然是 firmware INS_types.h |
| [G3 日志与可观测性设计](../G-harness/G3-logging.md) *(未启动,Wave 10)* | (无直接出边)| (G3 logsout 不依赖 contract diff;G3 日志的字段类型由 B1 直接保证)|
| **架构 v1 §17 Risk 1 / 6 / §10 silent enum drift**(架构层风险跟踪)| ⇢ | I3 §4.4.a / §4.4.b 实现把策略落到具体检测算法 |
| **INDEX.md 决策日志**(契约风险登记入口)| ⇢ | I3 §4.6 与 §4.4 的 `--allow-drift` 处理触发 INDEX 条目;但 I3 自身不直接写 INDEX(由 orchestrator / 运行者按 RULES §10 手工添加)|
| 全部 C / D / E / F / G 区下游(间接,变更回溯)| (变更回溯)| 任何 I3 检测的 firmware tip 漂移触发 RULES §10 受影响下游标记 |

整体而言:**I3 是 B4 策略的唯一实现侧;无强前置下游;影响 export pipeline 全部环节**(I4 / I5 / 实际 codegen 流程)。

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-07 | I3 author | 初稿(co-seal batch (B4, I3) Wave 5)|
| 2026-05-07 | orchestrator | shared Reviewer verdict=pass(per-doc 7/7 + cross-doc X-1..X-8 全部 met);frontmatter 升 reviewed。I3 contract_impact=no,INDEX 已记录无独立 contract impact 条目。非阻塞建议(下一轮迭代):(a) §4.7 worked example 中 `VehicleMode.MODE_HOLD` 与 B2 §4.4.5 类型 `FlightMode` 不一致(brief 指定值,后续以 `FlightMode.MODE_POSHOLD=4` 真实 enum 替换);(b) §4.7.2 `sizeof(FMS_Out_Bus)=312/316` 与 B1 §4.7 总宽 248B 对齐;(c) `--severity-threshold` 与 exit code 描述聚合于一处 |

## Self-check

- [x] frontmatter 完整,字段值合法
- [x] 上游文档全部存在且 status ≥ reviewed,**或**为本工作项所在 co-seal batch 的同批兄弟(架构 v1 已发布;A1 / A2 / A3 / A6 reviewed 2026-05-05;B1 / B2 / B3 reviewed 2026-05-07;B4 为 co-seal batch (B4, I3) 同批兄弟,allowed per RULES §6 第 2 项;§3 已注明 batch 名)
- [x] 退出条件逐条复核完成,每条均给出依据(§6 主退出条件表 12 项 + 任务 brief audit 追加表 + B4 矩阵覆盖证明表)
- [x] 引用路径全部可点击访问
- [x] 不存在 [RULES §5](../RULES.md) 禁则中的内容(无 .slx 截图;§4.4 / §4.5 伪代码块已标 `pseudo`,不是可执行 .m 或 .py;无 firmware 实现复述,只描述脚本侧设计;不重定义 bus/enum/PARAM 字段表 — 委托 B1/B2/B3;`*_types.h` 引用以路径占位 + `<pending hash>` per B1/B2/B3/B4 模式;§4.7 worked example 中 firmware path 与 SHA 是占位演示,非永久数据)
- [x] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记 — **N/A:本工作项 contract_impact=no**。理由:I3 是脚本设计文档,自身不定义任何 firmware-visible 契约字段(契约由 B1 / B2 / B3 拥有,契约保护策略由 B4 拥有)。I3 的实现错误会导致 B4 策略失效,但这是 contract-adjacent 而非 contract-defining。RULES §6 第 5 项的 INDEX 决策日志登记义务对 contract_impact=yes 的工作项激活;本工作项不需要,但 reviewer 应按高保护标准评审 I3(per §3 注与 §5 风险中的 "contract-adjacent" 声明)
- [x] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径 — **N/A:本工作项不创建 firmware 契约镜像产物**。I3 是**读** firmware 头文件的脚本,不输出契约镜像产物;镜像产物由 [B5 INS_Out_Bus 镜像同步流程](../B-contracts/B5-ins-bus-mirror.md) 与 [I2 Bus/enum 镜像脚本设计](I2-bus-enum-mirror.md) 生成。本文件 §3 已给出 firmware 路径(`FMT-Firmware/src/model/{plant,fms,control}/<vehicle>/lib/{Plant,FMS,Controller}_types.h` + `FMT-Firmware/src/model/ins/lib/INS_types.h`),commit hash 占位 `<pending hash>` 与 B1/B2/B3/B4 同期补齐
- [x] 下游影响已沿关系图识别完毕(§7 含 B4 协同 + I2 / I4 / I5 / B5 间接 + 架构 v1 / INDEX / 全部 C/D/E/F/G 区受变更回溯影响)
- [x] 文档不超出本工作项范围(无越权设计:bus/enum/PARAM 字段表留 B1/B2/B3;diff 策略 / 阈值 / 触发时机留 B4;codegen 配置留 I4;sim runner 留 I5;mirror 流程留 B5/I2;cmd_mask 位号留 D6/E1)
