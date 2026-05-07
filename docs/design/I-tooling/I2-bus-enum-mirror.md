---
work_item: I2
title: Bus/enum 镜像脚本设计
upstream: ["架构 v1", "A2", "B1", "B2", "B3"]
contract_impact: no
status: draft
authored_at: 2026-05-07
last_reviewed_at:
reviewer_verdict: none
---

# I2 Bus/enum 镜像脚本设计

## 1. 目的

设计一个**离线脚本**(以下称"镜像脚本",CLI 名暂定 `mirror.py`),把 FMT-Firmware 仓库的 `*_types.h` 头文件作为输入,**机械地**抽出 bus typedef、enum、parameter struct,产出 model-repo 侧的 Simulink Bus Object / `Simulink.IntEnumType` / `Simulink.Parameter` 与对应的 **YAML 镜像**两份产物。本文件锁定:**抽取规则、生成模板、人工 review 检查点、CLI 与失败模式**。它是 [B1](../B-contracts/B1-bus-inventory.md) / [B2](../B-contracts/B2-enum-inventory.md) / [B3](../B-contracts/B3-parameter-schema.md) 的**实现侧**(契约由 B 区定义,本文件只设计如何从 firmware tip 把契约自动镜像到模型仓);它是 [B5 INS_Out_Bus 镜像同步流程](../B-contracts/B5-ins-bus-mirror.md) 的执行体;它是 [I3 契约 diff 脚本设计](I3-contract-diff.md) 的输入产生方。

## 2. 范围

**在范围:**

- 镜像脚本的**架构与流水线**(text → C lexer/parser → AST → canonical model → render → 产物)
- 输入侧的**抽取规则**:bus typedef / enum / `*_PARAM_TYPE` / `*_EXPORT_TYPE` 的识别与字段提取规则,以及跳过/警告规则
- 输出侧的**双产物形态**:Simulink Bus Object(.sldd / .mat,主用)+ YAML 镜像(.yaml,供 I3 消费)
- C type → Simulink type **映射表**,以及 ambiguity(`int` 无显式宽度等)的处置
- **vehicle leaf 处理**:per-vehicle 输出策略(Phase 2 = multicopter only,呼应架构 v1 §16 与 [A5 §4.5](../A-architecture/A5-variant-strategy.md))
- **人工 review 检查点**:additive 自动合并 vs 重排/删除/类型变更走 manual review(呼应 audit F-32 / RULES §5)
- **CLI**(参数、退出码、`--help` 示例)
- **worked example**(audit F-18 要求每个 I 区设计文档必带):合成 `INS_types.h` 片段 → 一次调用 → YAML 输出 + header comment block
- **失败模式与限制**:padding/alignment 检测留给 B4/I3、macro-driven 类型别名 warn-only 等
- 与 B5 / I3 的**信息流耦合**

**不在范围(由其他工作项处理):**

- bus / enum / parameter 的**字段定义本身** — 由 [B1](../B-contracts/B1-bus-inventory.md) / [B2](../B-contracts/B2-enum-inventory.md) / [B3](../B-contracts/B3-parameter-schema.md) 处理;本文件只定义"如何把契约从 firmware 镜像出来",不复述任何字段
- **契约 diff 逻辑**(YAML 对比 / 报告格式 / fail 阈值)— 由 [B4 契约 diff 策略](../B-contracts/B4-contract-diff.md) 与 [I3 契约 diff 脚本设计](I3-contract-diff.md) 处理;本文件只承诺产出 I3 可消费的 YAML 形态
- **跨仓库 PR 工作流**(谁开 PR、commit 协调、auto-merge 规则)— 由 [B5 INS_Out_Bus 镜像同步流程](../B-contracts/B5-ins-bus-mirror.md) 处理;本文件只承诺 mirror 产物记录 firmware commit hash + 文件相对路径(per RULES §5)
- **codegen 配置**(ert.tlc 选项、`Tunable=on/off`、StorageClass)— 由 [I4 Codegen 配置脚本设计](I4-codegen-config.md) 处理
- **firmware 实际数值**:本设计文档定义**规则集**,不引用任何 firmware tip 数值;具体 firmware 路径用 `<pending hash>` 占位
- **可执行代码**:本文件只含标 `pseudo` 的伪代码与合成 worked example 文本(per RULES §5)

## 3. 依赖

| 上游 | 引用位置 | 用途 |
|---|---|---|
| 架构 v1 §5.1 / §10 / §11 / §14.1 / §14.2 / §17 risk 1 / risk 2 | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | §5.1 firmware 契约面是镜像源;§10 enum drift 风险是 review gate 的根因;§11 cross-vehicle leaf 决定 per-vehicle 输出;§14.1/14.2 bus/enum 必须有 canonical source;§17 risk 1 contract drift / risk 2 INS_Out_Bus drift 是脚本存在的根因 |
| [A2 命名与目录约定设计](../A-architecture/A2-naming-conventions.md) (status: reviewed, 2026-05-05) | A2 R-5.1 / R-5.2 / R-5.3 / E-5.1 / R-6.1 / R-6.2 / R-6.4 / R-7.1 / R-7.2 / R-7.3 / R-7.4 | bus 命名(契约 / 内部 / 字段)、enum 类型名 + 成员名、参数 struct 与 EXPORT 命名风格;镜像脚本必须按 A2 进行命名转换 |
| [B1 Bus 清单与 schema 设计](../B-contracts/B1-bus-inventory.md) (status: reviewed, 2026-05-07) | B1 §2 范围 / §4.x bus 字段元数据列定义 | I2 输出的 YAML 必须 field-for-field 等价于 B1 锁定的 16 个 bus schema 列(field name / Simulink type / byte width / unit / frame / encoding 等);I2 的 `jsonschema` 校验输入 = B1 列定义 |
| [B2 Enum 清单与数值锁定](../B-contracts/B2-enum-inventory.md) (status: reviewed, 2026-05-07) | B2 §2 范围 / §4.x enum 表(name / numeric / meaning / introduced-in) | I2 输出的 enum YAML 必须按 B2 列结构;数值锁定与 drift 检测要求由 B2 委派给 [B4](../B-contracts/B4-contract-diff.md) + I3 |
| [B3 Parameter schema 设计](../B-contracts/B3-parameter-schema.md) (status: reviewed, 2026-05-07) | B3 §2 范围 / §4.x PARAM/EXPORT 字段表 | I2 输出的 PARAM/EXPORT YAML 必须按 B3 列结构(field / type / unit / frame / classification 占位 / affects-bus-field / variant 标签) |
| [B5 INS_Out_Bus 镜像同步流程](../B-contracts/B5-ins-bus-mirror.md) | **Wave 5 sibling**(status: draft,本 wave 内将达 reviewed) | B5 定义抽取脚本的 I/O 契约(canonical 来源、版本标记、firmware commit hash + 路径必须随产物记录);I2 是 B5 的执行体。**注:B5 与 I2 在关系图 §5 中并非正式 co-seal,只是 wave-5 同 wave 兄弟;I2 forward-cite,初评时若 B5 仍未达 reviewed 须重审本节** |
| [I3 契约 diff 脚本设计](I3-contract-diff.md) | **Wave 5 sibling**(status: draft,本 wave 内将达 reviewed) | I3 消费 I2 产出的 YAML 进行 diff;I2 的 YAML schema 必须被 I3 兼容地读取(field-for-field 等价)。同 B5 注:并非正式 co-seal |
| RULES §5 / §6 | [链接](../RULES.md) | 镜像产物必须记录 firmware commit hash + 文件相对路径(本文件 §4.3 / §4.9);设计文档不得含 .slx 或可执行代码;伪代码标 `pseudo` |
| firmware 头文件路径(authoritative source) | `FMT-Firmware/src/model/{plant,fms,control,ins}/<vehicle>/lib/{Plant,FMS,Controller,INS}_types.h` @ `<pending hash>` | 本设计的输入面;FMT-Firmware 当前未在本环境挂载,故路径仅为约定占位,实际值由 B5 的 `firmware_commit_sha` 字段记录 |

注:B5 与 I3 是 wave-5 的同 wave 兄弟,关系图 §5 列出的正式 co-seal 组中并不包含 (I2, B5) 或 (I2, I3);本文件取得 reviewed 资格之前,B5/I3 应至少同时进入 reviewed,否则评审复核 §3。

## 4. 设计内容

### 4.1 脚本架构(流水线)

镜像脚本是一个**离线**(offline)Python 程序,在模型仓 CI 中运行,输入只读地访问 firmware 源码树,输出写入 `model/shared/bus/<vehicle>/`(及对应 `enum/`、`params/`)。它**不**执行 firmware 代码,**不**需要 ARM 工具链,**不**调用 MATLAB/Simulink runtime — 仅在 MATLAB 侧有一个可选的 helper(§4.7)把 YAML 转成 Simulink Bus Object。

ASCII 流水线:

```text
                                 +-------------------------+
firmware *_types.h (text)  -->  | 1. Reader              | 读取头文件文本,记录每文件的 SHA-1
                                 +-------------------------+
                                            |
                                            v
                                 +-------------------------+
                                 | 2. C Lexer/Parser      | pycparser(GCC fake-libc 预处理后的 AST)
                                 +-------------------------+
                                            |
                                            v
                                 +-------------------------+
                                 | 3. Extractor           | 仅识别本文件 §4.2 列出的目标节点
                                 +-------------------------+
                                            |
                                            v
                                 +-------------------------+
                                 | 4. Canonical Model     | language-agnostic 中间表示(IR);
                                 |  (Bus / Enum /         | 字段名、C type token、array length、
                                 |   PARAM / EXPORT IR)   | nested struct ref、source line/file
                                 +-------------------------+
                                            |
                              +-------------+-------------+
                              v                           v
                +---------------------+         +-----------------------+
                | 5a. YAML Renderer  |         | 5b. (optional) MATLAB |
                |   写出 .yaml 镜像   |         |   helper:YAML →       |
                |   供 I3 消费        |         |   Simulink Bus Object |
                +---------------------+         +-----------------------+
                              |                           |
                              v                           v
                model/shared/bus/<vehicle>/*.yaml    *.sldd / *.mat
                model/shared/enum/<vehicle>/*.yaml
                model/shared/params/schemas/<vehicle>/*.yaml
```

声明:这是 **OFFLINE** 脚本,在模型仓 CI 中调用;它不执行 firmware 代码、不链接 firmware 二进制、不依赖目标 MCU 工具链。"firmware tip → mirror" 的原子性由 §4.6 review gate 保证,而非脚本本身保证。

### 4.2 Input 抽取规则

镜像脚本对 firmware `*_types.h` 仅识别下列节点;其他节点跳过(必要时按 §4.10 警告)。

#### 4.2.1 Bus 类型(typedef struct)

**触发模式**:`typedef struct <opt_tag> { ... } <Name>_Bus;`(C99 typedef)。

**来源文件清单**:
- `FMT-Firmware/.../FMS_types.h` @ `<pending hash>`
- `FMT-Firmware/.../Controller_types.h` @ `<pending hash>`
- `FMT-Firmware/.../Plant_types.h` @ `<pending hash>`
- `FMT-Firmware/.../INS_types.h` @ `<pending hash>` (B5 单列)

**逐字段提取**:
1. 字段 C 名(原样,后续按 §4.3 命名规则转换)
2. C 基本类型 token(`uint8_t` / `uint16_t` / `uint32_t` / `int8_t` / `int16_t` / `int32_t` / `float` / `double` / `bool` / `char`)
3. 数组维度(`T name[N]` → `dims = [N]`;多维 `T name[M][N]` → `dims = [M, N]`,**只展开静态长度**)
4. 嵌套 typedef 引用(`Other_Bus inner;` 或 `Waypoint wp;` → 记录依赖,后续生成 Bus reference)
5. 字段所在源文件 + 行号(供 review gate 报告)
6. 同行尾注释(`/* unit: m, frame: NED */` 形式 — 若存在,作为 hint 透传给 YAML;**不**自动校正,以人工 review 为准)

#### 4.2.2 Enum 类型

**触发模式**:`typedef enum { MEMBER1 = 0, MEMBER2 = 1, ... } <Name>;` 或匿名 `enum { ... };` 关联到 `typedef`。

**逐成员提取**:
1. 成员名(原样)
2. 显式数值(若有 `= <int>`)或隐式数值(连续编号,从 0 起或上一显式值 +1)— **始终在 IR 中存为绝对整数**,不保留隐式形式,以利下游 diff
3. 同行尾注释作为 meaning hint
4. enum 底层整型(若 firmware 用 `__attribute__((packed))` 或 GCC `enum` 默认 `int` — 见 §4.4 ambiguity 处置)

#### 4.2.3 Parameter / Export struct

**触发模式**:`typedef struct { ... } FMS_PARAM_TYPE;` / `CONTROL_PARAM_TYPE` / `PLANT_PARAM_TYPE`,以及对应 `FMS_EXPORT_TYPE` / `CONTROL_EXPORT_TYPE` / `PLANT_EXPORT_TYPE`(架构 v1 §5.1.2 + A2 R-7.1 / R-7.2)。

**逐字段提取**:同 §4.2.1。额外:
- `*_PARAM_TYPE`:每字段需在 YAML 中保留 `classification: <unknown|tunable|inlined>` 占位(由 B3 §4.2 分类方法在人工 review 阶段填实);I2 不自动判定。
- `*_EXPORT_TYPE`:`period`(scalar)与 `model_info[]`(array of struct)字段必须存在;若缺失,§4.10 标 hard failure。

#### 4.2.4 跳过与警告规则

| 节点 | 处置 | 退出码 |
|---|---|---|
| 宏定义 `#define FOO 42` | 跳过(若数值用于 enum/array 长度,先经 pycparser 预处理展开;展开失败 → 警告) | 1 (warn) |
| 匿名 union `union { ... }`(无 typedef 名) | 跳过 + 警告(可能丢字段语义) | 1 (warn) |
| Bitfield `uint32_t flag : 1;` | **默认拒绝**,exit 2;**例外**:`INS_Flag` 由 B1 在 schema 中显式登记为 bitfield,§4.2 抽取规则下白名单允许(需 `--allow-bitfield <Name>` 显式 opt-in 或 white-list config) | 2 (hard) / 0 (whitelisted) |
| Function pointer 字段 | 跳过 + 警告(契约 bus 中不应出现) | 1 (warn) |
| 匿名嵌套 struct(非 typedef) | 展平到外层字段;若展平后字段名冲突 → exit 2 | 0 / 2 |
| `enum` 成员名为 `RESERVED` / `_PADDING` | 保留但标 `reserved: true`;不参与 B4 数值锁定 | 1 (warn,提醒 reviewer) |
| 注释驱动的字段重命名(只改 comment) | 不可检测(§4.10 限制) | — |
| Macro-driven 类型别名(`typedef MY_TYPE my_t`) | warn-only;若 `MY_TYPE` 为 §4.4 表内已知映射,则正常处理;否则记录到 review 报告并 exit 1 | 1 (warn) |

### 4.3 Output 生成规则(双产物)

镜像脚本对每个 IR 节点同时产出**两种**形态。两者必须 field-for-field 等价。

#### 4.3.1 主产物:Simulink Bus / DataDictionary / Parameter

| IR | 产物形态 | 产物路径(以 multicopter 为例) |
|---|---|---|
| Bus IR | `Simulink.Bus` 对象 + `Simulink.BusElement` 数组,装入 `.sldd` 或 `.mat` | `model/shared/bus/multicopter/<bus_name>.sldd` |
| Enum IR | `Simulink.IntEnumType` 类定义 (或等价 `.sldd` enum) | `model/shared/enum/multicopter/<enum_name>.sldd` |
| PARAM IR | `Simulink.Parameter` + `Simulink.Bus`(嵌套结构);默认 `StorageClass=ImportedExtern`(I4 决定最终值) | `model/shared/params/schemas/multicopter/<struct_name>.sldd` |
| EXPORT IR | 仅 schema 占位;实际产出由代码生成驱动 | `model/shared/params/schemas/multicopter/<struct_name>_export.yaml`(只 YAML;EXPORT 不需要 Simulink Bus,因为它是 codegen 产物形态) |

注:实际把 YAML 转 Simulink artifact 由 §4.7 的 MATLAB 侧 helper 完成;镜像脚本本身只产出 YAML,使流水线在无 MATLAB 的 Linux CI runner 上也可运行。

#### 4.3.2 次产物:YAML 镜像(I3 消费)

每个 IR 节点产出一个 `.yaml` 文件,顶部带 header comment block(per B5 §4.3):

```yaml
# === MIRRORED FROM FMT-FIRMWARE — DO NOT HAND EDIT ===
# firmware_commit_sha: <pending hash>
# firmware_path: src/model/ins/multicopter/lib/INS_types.h
# extracted_at: 2026-05-07T00:00:00Z
# extractor: mirror.py v0.1
# vehicle: multicopter
# review_gate: pending
# === END HEADER ===
kind: bus              # | enum | param | export
name: INS_Out_Bus
underlying_int: null   # 仅 enum 使用
fields:
  - name: timestamp
    type: uint32
    dims: []
    unit: ms           # hint,人工 review 锁定;A2 E-10.1 firmware-legacy 例外
    frame: null
    encoding: null
    source_line: 17
total_byte_width_hint: 92   # natural-alignment best-effort;真值由 B4/I3 检测
```

**等价性约束**:Simulink Bus / Enum / Parameter 的字段顺序、字段名(经 §4.3.3 转换)、Simulink 类型(经 §4.4 映射)必须与 YAML 文件 field-for-field 一致;违反则 §4.6 review gate 拒绝。I3 仅消费 YAML(便于在 Linux CI 跑 diff),Simulink artifact 在 model authoring 侧消费。

#### 4.3.3 命名转换规则(A2 ↔ firmware 对齐)

镜像脚本按 A2 规则做命名转换:

| 对象 | firmware 形态(C 侧观察) | 模型仓侧目标(per A2) | 转换规则 |
|---|---|---|---|
| Bus typedef | `<Name>_Bus`(已 PascalCase + `_Bus` 后缀) | A2 R-5.1 沿用 | **不变换**(契约 bus 名固化) |
| Bus 字段名 | 一般为 `lower_snake_case`;firmware 已固化字段(如 `timestamp`)无后缀 | A2 R-5.3 要求单位/坐标系后缀(`_mps`、`_rad` 等) | **不变换**(A2 E-5.1 例外:firmware-legacy 字段名优先;只透传 hint,B1 schema 锁单位) |
| enum 类型名 | 多为 PascalCase;偶有 `_E` 后缀 | A2 R-6.1 PascalCase 无后缀 | 若发现 `_E` 后缀 → **warn**(exit 1)请求人工裁定;不自动改名(契约 enum 名固化) |
| enum 成员名 | 多为 `UPPER_SNAKE_CASE`,带或不带类型短前缀 | A2 R-6.2 要求带类型短前缀 | **不变换**(A2 E-5.1 同款例外延伸:契约 enum 已固化的成员名优先;若 firmware 未带前缀,B2 已对齐表豁免;不自动加前缀) |
| PARAM/EXPORT struct 名 | `FMS_PARAM_TYPE` / `FMS_EXPORT_TYPE`(C 侧 `_TYPE` 后缀) | A2 R-7.1/R-7.2 模型仓侧引用 `FMS_PARAM` / `FMS_EXPORT`(无 `_TYPE`) | **去除 `_TYPE` 后缀**(C typedef 习惯 vs Simulink 引用习惯) |
| PARAM/EXPORT 字段名 | `lower_snake_case` | A2 R-7.3 同 | **不变换** |

**总原则**:契约对象一旦在 firmware 固化,模型仓侧**不**重命名(per A2 E-5.1);命名转换仅限 typedef `_TYPE` 后缀剥离。任何脱出此规则的转换都触发 §4.6 manual review。

### 4.4 C type → Simulink type 映射

| C type | Simulink type | 备注 |
|---|---|---|
| `uint8_t` | `uint8` | 直接映射 |
| `uint16_t` | `uint16` | 直接映射 |
| `uint32_t` | `uint32` | 直接映射 |
| `uint64_t` | `uint64` | 契约 bus 不应出现;若出现 → warn |
| `int8_t` | `int8` | 直接映射 |
| `int16_t` | `int16` | 直接映射 |
| `int32_t` | `int32` | 直接映射 |
| `int64_t` | `int64` | 契约 bus 不应出现;若出现 → warn |
| `float` | `single` | 32-bit IEEE 754 |
| `double` | `double` | 64-bit IEEE 754 |
| `bool` (`stdbool.h`) | `boolean` | 直接映射;隐含 1 byte |
| `char` | `int8` | 罕见;契约 bus 不应有字符串 |
| nested `<Other>_Bus` | Bus reference | YAML 写 `type: ref`,`ref: <Other>_Bus` |
| array `T[N]` | scalar `T` + `Dimensions = N` | `dims: [N]` |
| array `T[M][N]` | scalar `T` + `Dimensions = [M N]` | `dims: [M, N]` |
| enum (`typedef enum {...} E;`) | `Simulink.IntEnumType` ref(YAML `type: enum`,`ref: E`) | underlying int 由 §4.2.2 / §4.4 ambiguity 节决定 |
| `int`(无显式宽度) | **warn + map to `int32`** | ARM Cortex-M ABI 默认 `int` = 32-bit;exit code 1;rationale 写入 review 报告 |
| `unsigned`(无显式宽度) | warn + map to `uint32` | 同上 |
| `enum X` 无显式 underlying | warn + map to `int32` | GCC 默认 enum 整型 = `int`;若 firmware 使用 `__attribute__((packed))` enum,B2 在 enum 表中显式声明 underlying 整型,镜像脚本以 B2 声明为准 |
| `size_t` / `ssize_t` / `ptrdiff_t` | exit 2(hard failure) | 平台依赖,不应出现于契约面 |
| pointer `T*` | exit 2(hard failure) | 契约 bus 不允许指针 |
| function pointer | exit 2(hard failure) | 同上 |

**Ambiguity 总则**:一旦遇到平台依赖类型(`int`、`long`、`size_t` 等),脚本**绝不**静默选择;必须 warn(exit 1)或 hard failure(exit 2),并把决策写入 review 报告。

### 4.5 Variant / vehicle leaf 处理

#### 4.5.1 Per-vehicle 输出

firmware 头文件树按 `<vehicle>` 分桶(`multicopter` / `fixwing` / `vtol` / `boat` / `car` / `submarine` / `template`,per A2 R-2.2)。镜像脚本对每个 vehicle 独立产出:

```text
model/shared/bus/<vehicle>/*.yaml             (+ .sldd via §4.7 helper)
model/shared/enum/<vehicle>/*.yaml
model/shared/params/schemas/<vehicle>/*.yaml
```

不**合并**多 vehicle 的 schema 到单一文件;cross-vehicle 同名 bus 即使字段相同也分别落盘,以便:
- 任一 vehicle leaf 新增 vehicle-specific 字段时不影响其他 vehicle 的 mirror
- 跨 vehicle drift 在 §4.6 review gate 显性出现(非隐式 mask)

#### 4.5.2 Phase 2 = multicopter only

按架构 v1 §16 与 [A5 §4.5](../A-architecture/A5-variant-strategy.md),Phase 2 仅交付 multicopter。镜像脚本必须支持:
- **single-vehicle 模式**(默认):`--vehicle multicopter` — 仅扫描 multicopter 头文件;Phase 2 唯一调用形态
- **multi-vehicle batch 模式**:`--vehicle all` — 遍历 firmware 所有 vehicle 目录,逐一生成。Phase 2 **不**触发,但脚本必须支持以便 Phase 5 vehicle 扩展时不需要重写

batch 模式要求各 vehicle 的产物彼此独立;**任一 vehicle 失败不影响其他**(逐 vehicle 的 exit code 收敛到 batch summary;批量调用的整体 exit = max(各 vehicle exit code))。

### 4.6 人工 review 检查点

per audit F-32 / RULES §5(镜像产物必须有 review 闸口防止"silently mirrored"),镜像脚本运行后**强制**进入 review gate。该 gate **不是**镜像脚本的代码逻辑,而是**工作流约定**;脚本只产出便于 review 的 artifact + 报告。

#### 4.6.1 每次跑后的产物

| 产物 | 形态 | 用途 |
|---|---|---|
| YAML mirror 文件 | per §4.3.2 | I3 diff 输入 |
| `mirror_report.json` | 结构化报告 | review gate 可读 |
| `mirror_report.md` | 人类可读总结 | review reviewer 用 |
| `mirror_diff_summary.md` | 与上轮 mirror 的字段级 diff(脚本内置 diff,粗粒度 — I3 跑细粒度) | review reviewer 第一眼 |

#### 4.6.2 三类 review 闸口

| 变更类型 | 示例 | 闸口 |
|---|---|---|
| **Auto-merge gate**(允许 CI 自动合并) | (a) 在 struct 末尾追加新字段;(b) 在 enum 末尾追加新成员且数值连续递增;(c) firmware 注释更新但类型/名/数值不变 | CI 通过 + 报告中 `category=additive_safe` → reviewer 仅追认 |
| **Manual review gate**(必须人工裁定后才能合并) | (a) 字段顺序变化;(b) 字段类型变化;(c) 字段重命名(同位置字段名变);(d) enum 数值非末尾插入或重新分配;(e) 字段删除;(f) 数组长度变化;(g) bitfield 例外白名单触发;(h) 任一 §4.4 ambiguity warn(`int` 无宽度等);(i) anonymous union 警告;(j) cross-vehicle drift(同名 bus 在不同 vehicle 字段不一致) | 报告 `category=needs_review` → reviewer 必须 sign-off |
| **Hard failure gate**(脚本 exit 2,根本无法产出 mirror) | parse error、不支持构造(指针、`size_t`)、必备字段缺失(EXPORT 无 `period` / `model_info[]`)、命名违反 A2 且无法自动豁免 | 阻断 CI;reviewer 必须先解决 hard failure 才能进入上面两档 |

#### 4.6.3 review checklist(人工执行)

reviewer 在 manual gate 收到 `mirror_diff_summary.md` 后必须逐条核对:

1. 是否有**意外的 enum 位插入**(尤其 mode/state/error,呼应 [B2](../B-contracts/B2-enum-inventory.md) drift critical 风险)
2. 是否有**struct 字段重排**(直接破坏字节布局)
3. 是否有**新字段语义未知**(reviewer 必须查 firmware commit message 或问 firmware owner)
4. 是否有 anonymous union / bitfield 警告未澄清
5. 是否有**类型变化**(如 `uint16_t → uint32_t`,字段加宽 — 应升级到 contract review,不仅 manual gate)
6. 是否有**字段从一个 bus 移到另一个 bus**(跨 bus 重组)— pycparser 无法跨文件检测,需 reviewer 由 commit history 看出
7. cross-vehicle drift:multicopter 有但 fixwing 无 / 字段不同 — 是否符合 vehicle-specific 设计意图

review 通过后,reviewer 在 `mirror_report.md` 末尾追加签字行;CI hook 检查签字行存在才允许 mirror artifact 入仓。

### 4.7 实现注记(非可执行)

> 标记 `pseudo`:本节仅描述设计选型,**不**包含可执行代码。

- **语言**:Python 3.11+(Linux CI 可用,无 MATLAB 依赖)
- **C parser**:[`pycparser`](https://github.com/eliben/pycparser) — 纯 Python 实现;需配合 fake-libc 头(pycparser 自带)对 firmware 头做预处理
- **YAML 输出**:`pyyaml`(`yaml.safe_dump`,`sort_keys=False`,确保字段顺序写出)
- **JSON Schema 校验**:`jsonschema` — 镜像产物 YAML 在写盘前根据 B1/B2/B3 §<schema> 节给出的 JSON Schema 校验;违反 → exit 2
- **logging**:Python 标准 `logging`,INFO/WARN/ERROR 三级;ERROR 即 exit ≥ 2
- **MATLAB 侧 YAML → Simulink Bus helper**:可选 helper(本设计 §2 范围:I2 自身只到 YAML;helper 由 MATLAB R2025b 内置 `Simulink.importExternalCTypes` 或脚本化构造 `Simulink.Bus` / `Simulink.IntEnumType` 完成)。该 helper 设计放在 [I1 仓库初始化脚本设计](I1-init-script.md) 或 I4 的相邻产物中,本文件不展开。
- **跨平台**:仅 Linux CI 路径必须可跑;macOS/Windows best-effort(pycparser 跨平台,YAML 跨平台,无系统调用依赖)
- **firmware repo 路径传入**:CLI `--firmware <path>`;脚本不假设 firmware 与模型仓共享父目录
- **commit hash 由调用方传入**:`--commit <sha>`;脚本不自己 `git rev-parse`(避免假设 firmware 是 git 子模块或 worktree;调用方负责把 firmware repo 状态变成 hash)

伪代码流水线(`pseudo`):

```pseudo
function mirror(firmware_path, commit_sha, vehicle, out_dir, strict_naming, dry_run):
    headers = locate_headers(firmware_path, vehicle)         # §4.2 来源清单
    asts    = [pycparser.parse(preprocess(h)) for h in headers]
    irs     = []
    for ast in asts:
        for node in walk(ast):
            if matches_bus_typedef(node):    irs.append(extract_bus(node))
            elif matches_enum_typedef(node): irs.append(extract_enum(node))
            elif matches_param_struct(node): irs.append(extract_param(node))
            elif matches_export_struct(node):irs.append(extract_export(node))
            elif is_skippable(node):          continue
            else:                             warn_or_fail(node)        # §4.2.4
    for ir in irs:
        validate_against_schema(ir, jsonschema_for(ir.kind))             # §4.7
        ir.yaml = render_yaml(ir, header_block(commit_sha, vehicle))     # §4.3.2
        if not dry_run: write(out_dir / path_for(ir), ir.yaml)
    report = build_report(irs, prev_irs=load_prev(out_dir))              # §4.6.1
    write(out_dir / "mirror_report.{json,md}", report)
    return summary_exit_code(report)                                     # §4.8
```

### 4.8 CLI

```text
mirror.py --firmware <path> --commit <sha> --vehicle <name> --out <dir>
          [--strict-naming] [--dry-run] [--allow-bitfield <Name>...]
          [--prev-mirror <dir>]
```

| 参数 | 类型 | 必选 | 说明 |
|---|---|---|---|
| `--firmware <path>` | path | 是 | FMT-Firmware repo 根目录;脚本自此向下找 `src/model/<module>/<vehicle>/lib/<Module>_types.h` |
| `--commit <sha>` | string | 是 | firmware tip commit SHA;写入 mirror header(per B5 §4.3 / RULES §5);脚本不自己解析 git |
| `--vehicle <name>` | string | 是 | `multicopter` / `fixwing` / `vtol` / `boat` / `car` / `submarine` / `all`(per A2 R-2.2) |
| `--out <dir>` | path | 是 | 模型仓侧目标目录(建议 `model/shared/bus/<vehicle>/` 等子树的根) |
| `--strict-naming` | flag | 否 | 把 §4.3.3 的 warn 升级为 hard failure(用于本仓 CI 主分支);Phase 2 默认 off |
| `--dry-run` | flag | 否 | 不写盘,只产出 report;CI smoke test 用 |
| `--allow-bitfield <Name>` | repeatable | 否 | 显式白名单某个 typedef(默认仅允许 `INS_Flag`);触发 §4.2.4 例外路径 |
| `--prev-mirror <dir>` | path | 否 | 上轮 mirror 目录;用于产出 `mirror_diff_summary.md`(§4.6.1);省略时跳过 diff 摘要 |

#### Exit codes

| code | 含义 | 触发条件示例 |
|---|---|---|
| `0` | clean mirror produced | 全部 IR 通过 §4.2 / §4.4 / §4.7 校验,产物落盘 |
| `1` | mirror produced with warnings | `int` 无宽度、anonymous union skip、注释更新 hint、enum 成员名 `RESERVED`、macro 别名未识别 — reviewer 必须看 |
| `2` | hard failure(no mirror) | parse error、不支持构造(指针 / `size_t` / 非白名单 bitfield)、命名违反且 `--strict-naming`、`*_EXPORT_TYPE` 缺 `period`/`model_info[]`、jsonschema 校验失败 |
| `3` | tool / environment error | `pycparser` 未安装、firmware path 不存在、out dir 无权写、commit SHA 格式非法等(与契约本身无关) |

#### `--help` 示例

```text
$ mirror.py --help
mirror.py — FMT-Model-2025b firmware-types mirror

USAGE
  mirror.py --firmware <path> --commit <sha> --vehicle <name> --out <dir>
            [--strict-naming] [--dry-run] [--allow-bitfield <Name>...] [--prev-mirror <dir>]

EXAMPLE
  mirror.py --firmware /work/FMT-Firmware --commit abc1234 \
            --vehicle multicopter --out model/shared/bus/multicopter \
            --prev-mirror model/shared/bus/multicopter

EXIT CODES
  0 = clean
  1 = warnings (mirror produced, reviewer must inspect)
  2 = hard failure (no mirror)
  3 = tool/environment error
```

### 4.9 Worked example(audit F-18 必备)

#### (a) 输入:合成 `INS_types.h` 片段

> 仅为示例;**不**取自任何真实 firmware tip 数值。文件路径占位 `FMT-Firmware/src/model/ins/multicopter/lib/INS_types.h @ <pending hash>`。

```c
/* synthetic INS_types.h fragment — not real firmware content */
#include <stdint.h>
#include <stdbool.h>

typedef enum {
    INS_STATUS_INIT  = 0,
    INS_STATUS_ALIGN = 1,
    INS_STATUS_VALID = 2,
    INS_STATUS_FAULT = 3,
} INS_Status;

typedef struct {
    uint32_t   timestamp;          /* ms since boot */
    float      pos_ned[3];         /* m, NED */
    float      vel_ned[3];         /* m/s, NED */
    float      quat[4];            /* w,x,y,z */
    float      ang_rate_b[3];      /* rad/s, body */
    INS_Status status;
    uint16_t   ins_flag;           /* bitfield, see INS_Flag (mirrored separately) */
    bool       reserved[2];
} INS_Out_Bus;
```

#### (b) 命令

```text
$ mirror.py --firmware /tmp/fw --commit abc1234 \
            --vehicle multicopter \
            --out model/shared/bus/multicopter
```

#### (c) 输出:YAML mirror(供 I3 消费)

文件 `model/shared/bus/multicopter/INS_Out_Bus.yaml`:

```yaml
# === MIRRORED FROM FMT-FIRMWARE — DO NOT HAND EDIT ===
# firmware_commit_sha: abc1234
# firmware_path: src/model/ins/multicopter/lib/INS_types.h
# extracted_at: 2026-05-07T00:00:00Z
# extractor: mirror.py v0.1
# vehicle: multicopter
# review_gate: pending
# === END HEADER ===
kind: bus
name: INS_Out_Bus
fields:
  - name: timestamp
    type: uint32
    dims: []
    unit_hint: ms          # firmware comment;A2 E-10.1 firmware-legacy 例外
    frame: null
    source_line: 12
  - name: pos_ned
    type: single
    dims: [3]
    unit_hint: m
    frame: NED
    source_line: 13
  - name: vel_ned
    type: single
    dims: [3]
    unit_hint: m/s
    frame: NED
    source_line: 14
  - name: quat
    type: single
    dims: [4]
    unit_hint: null
    frame: null
    encoding_hint: "w,x,y,z"
    source_line: 15
  - name: ang_rate_b
    type: single
    dims: [3]
    unit_hint: rad/s
    frame: body
    source_line: 16
  - name: status
    type: enum
    ref: INS_Status
    dims: []
    source_line: 17
  - name: ins_flag
    type: uint16
    dims: []
    encoding_hint: "bitfield ref INS_Flag"
    source_line: 18
  - name: reserved
    type: boolean
    dims: [2]
    source_line: 19
total_byte_width_hint: 60   # natural-alignment best-effort; B4/I3 verifies
```

文件 `model/shared/enum/multicopter/INS_Status.yaml`:

```yaml
# === MIRRORED FROM FMT-FIRMWARE — DO NOT HAND EDIT ===
# firmware_commit_sha: abc1234
# firmware_path: src/model/ins/multicopter/lib/INS_types.h
# extracted_at: 2026-05-07T00:00:00Z
# extractor: mirror.py v0.1
# vehicle: multicopter
# review_gate: pending
# === END HEADER ===
kind: enum
name: INS_Status
underlying_int: int32       # GCC default; warn-eligible per §4.4 ambiguity
members:
  - name: INS_STATUS_INIT
    value: 0
  - name: INS_STATUS_ALIGN
    value: 1
  - name: INS_STATUS_VALID
    value: 2
  - name: INS_STATUS_FAULT
    value: 3
```

加上 `mirror_report.md` 中 reviewer-relevant 摘要:`category=needs_review`,因为脚本同时报告 `INS_Status` underlying int 落到 ambiguity warn 轨道(exit 1),**reviewer 必须**裁定 underlying 是否就是 `int32`(否则需要 `--strict-naming` + B2 显式声明)。

#### (d) 接下来 I3 跑的 check(一行清单)

I3 (per [I3](I3-contract-diff.md) wave-5 sibling)消费上述 YAML,运行 [B4](../B-contracts/B4-contract-diff.md) 列出的覆盖项:

- bus 字段顺序与字节宽度 vs 上一锁定 mirror = identical?
- enum 数值 vs 上一锁定 mirror = identical?(any insertion / renumber → fail)
- `*_PARAM` / `*_EXPORT` 字段顺序与类型 = identical?
- `*_EXPORT.period` 数值不变 / `model_info[]` 长度不变?
- 符号存在性:`Plant_init` / `FMS_init` / `Controller_init` / `*_step` 在 codegen artifact 中可见?

### 4.10 失败模式与限制

| 限制 | 后果 | 处置 |
|---|---|---|
| 脚本**不能**检测 padding / alignment 差异 | mirror 给的 `total_byte_width_hint` 可能与 firmware 编译器实际布局不一致 | 留给 [B4](../B-contracts/B4-contract-diff.md) + [I3](I3-contract-diff.md) 在 export 阶段做 byte-width check;mirror 显式标 `(B4 to verify)` |
| 脚本**不能**检测 macro-driven 类型别名 | `typedef MY_TYPE my_t;`,若 `MY_TYPE` 在另一 header `#define MY_TYPE uint16_t` 处展开 — pycparser 预处理通常能展开,但跨多层条件编译可能失败 | warn-only(exit 1);记录到 review 报告;reviewer 决定是否手动 patch |
| 脚本**不能**检测 comment-only 字段重命名 | firmware 改注释,字段名/类型不变 — IR 完全相同,diff = empty | 不在 I2 范围;若需要,留给 firmware-side review 流程或后续工具增强 |
| 脚本**不能**检测跨 bus 字段迁移 | 同名字段从 `BusA` 移到 `BusB` — 镜像脚本只看单个 typedef,看不到"消失字段重新出现" | 由 §4.6.3 reviewer checklist 第 6 条用 commit history 兜底 |
| **Cross-vehicle drift** | leaf vehicle 添加了 vehicle-specific 字段,导致同名 bus 在不同 vehicle 字段不同 | per-vehicle 输出策略(§4.5.1)使其在文件层面**显性可见**;再经 §4.6 review gate 第 7 条人工核对;镜像脚本本身不裁决 |
| **Bitfield 默认拒绝**导致 `INS_Flag` 等已知 bitfield 也被拒 | hard failure | 通过 `--allow-bitfield INS_Flag` 显式 opt-in;长期 white-list 由 B1 在 schema 中登记 |
| **`int` 无显式宽度** | warn-only,默认 `int32`(ARM Cortex-M ABI) | reviewer 必须查 firmware 是否针对其他 ABI;`--strict-naming` 模式下升级为 hard failure |

### 4.11 与 Wave 5 兄弟的耦合

#### 4.11.1 与 [B5 INS_Out_Bus 镜像同步流程](../B-contracts/B5-ins-bus-mirror.md)(Wave 5 sibling,status: draft → reviewed)

B5 定义"镜像同步"的契约 I/O:
- canonical 来源(firmware 单向 mirror vs 共享 schema 子模块)
- 镜像产物必须记录 `firmware_commit_sha` + `firmware_path`(per RULES §5)
- 抽取脚本的输入输出形态 + 版本标记策略

I2 是 B5 的**执行体**:本文件 §4.3.2 / §4.9 的 header comment block 形式与字段清单与 B5 §4.3 协议保持一致。**注**:B5 是 wave-5 同 wave 兄弟,**不**是关系图 §5 列出的正式 co-seal;I2 forward-cite 时明示其 status: draft — 若 B5 在本 wave 评审时仍是 draft,本 §4.11.1 的具体字段名(`firmware_commit_sha` / `firmware_path` / `extracted_at`)必须随 B5 reviewed 版定稿值复核。

#### 4.11.2 与 [I3 契约 diff 脚本设计](I3-contract-diff.md)(Wave 5 sibling,status: draft → reviewed)

I3 是 I2 产出的**消费者**:
- I3 读取 I2 写出的 YAML(§4.3.2 形态)
- I3 的 diff 字段集 = I2 YAML 的字段集(field-for-field 等价)
- I3 失败响应规则(B4 退出条件)对应 I2 review gate(§4.6.2)三档分级中的"Manual" / "Hard"档

I2 与 I3 的边界:**I2 给"镜像快照",I3 给"两个快照之间的 diff"**。I2 自身 §4.6.1 内置的 `mirror_diff_summary.md` 仅是粗粒度 diff(变更类别分桶),细粒度字段级 diff 报告由 I3 出。**注**:同 §4.11.1,wave-5 同 wave 兄弟 ≠ co-seal;若评审 I2 时 I3 仍是 draft,本节字段名约定须与 I3 reviewed 版重核。

#### 4.11.3 与 [B1](../B-contracts/B1-bus-inventory.md) / [B2](../B-contracts/B2-enum-inventory.md) / [B3](../B-contracts/B3-parameter-schema.md)(已 reviewed)

I2 输出的 YAML schema **必须**与 B1/B2/B3 §4.x 列定义 field-for-field 等价(本文件 §4.3.2 / §4.7 jsonschema 校验环节直接消费 B1/B2/B3 列定义)。本文件不复述 B 区字段;若 B 区任一文档变更,I2 的 jsonschema 来源同步更新,本文件视为 `Affected by upstream change`(per RULES §10)。

## 5. 已知风险与悬而未决问题

- **B5 / I3 wave-5 sibling 状态依赖**
  - 影响:B5 / I3
  - 处置:本文件取得 reviewed 资格之前,B5 与 I3 须**至少同时**进入 reviewed;若评审 I2 时其中任一仍为 draft,则 §3 / §4.11 标注复核,reviewer 须确认 forward-cite 的字段名(`firmware_commit_sha` / `firmware_path` / `extracted_at`)与 B5 / I3 reviewed 版一致。

- **pycparser 对真实 firmware tip 的覆盖度**
  - 影响:I2 自身可执行性 — pycparser 不支持 GCC 全套扩展(`__attribute__((aligned))` / `__asm__` / 复杂宏)
  - 处置:open;首次实跑(实现阶段)由 fake-libc 头 + 自定义预处理脚本兜底;若 firmware tip 触发不可解析构造 → exit 2,迫使 firmware 侧或本脚本修补,而非静默 mirror 错。

- **enum underlying int 推断的不确定性**
  - 影响:B2 数值锁定 — GCC 默认 enum = `int`(32-bit),但若 firmware 用 `__attribute__((packed))` 则可能 1-byte
  - 处置:I2 给 ambiguity warn(exit 1);权威决策权移交 [B2](../B-contracts/B2-enum-inventory.md) — B2 在 enum 表中显式登记 underlying 整型,镜像脚本以 B2 声明为准。

- **MATLAB 侧 YAML → Simulink Bus helper 归属**
  - 影响:I1 / I4 — 该 helper 是 Linux CI 不需要、MATLAB 侧需要的桥
  - 处置:open;归属候选 [I1 仓库初始化脚本](I1-init-script.md) 或 I4 codegen 配置;Phase 2 实现阶段决定。本文件仅在 §4.3.1 / §4.7 占位,不裁决。

- **跨 vehicle drift 的可执行检测**
  - 影响:Phase 2 不触发(只有 multicopter),Phase 5 才暴露
  - 处置:Phase 5 启动前由 §4.6.3 review checklist 第 7 条人工兜底;若届时 vehicle 数量 >2,本文件视为 `Affected by upstream change` 重审 §4.5。

- **`--strict-naming` 在 Phase 2 默认 off 的过渡风险**
  - 影响:firmware 已固化命名(A2 E-5.1 例外)与 A2 R-5.3 / R-7.4 的单位后缀义务可能持续不一致,但镜像脚本不该报错
  - 处置:已规避 — A2 E-5.1 已声明 firmware-legacy 字段名豁免;镜像脚本**不**改 firmware-legacy 字段名,仅在 review 报告标 hint。

## 6. 退出条件复核

对照 [`00-design-plan.md`](../00-design-plan.md) §4.I 中本工作项"退出条件"原文:**"从 firmware *_types.h 抽取规则、生成模板、人工 review 点"**(audit F-18 同时要求 I 区每文档必带 worked example)。

| # | 退出条件原文 | 本文档依据 | 状态 |
|---|---|---|---|
| 1 | 从 firmware `*_types.h` **抽取规则** | §4.2(逐节点抽取规则,含 §4.2.4 跳过/警告表) + §4.4(C type → Simulink type 映射 + ambiguity 处置) | 满足 |
| 2 | **生成模板**(YAML mirror + Simulink Bus 双形态) | §4.3.1(主产物形态表 + 路径规则) + §4.3.2(YAML 模板含 header comment block + jsonschema-aligned fields) + §4.3.3(命名转换规则) + §4.9 worked example 落实模板 | 满足 |
| 3 | **人工 review 点** | §4.6.1(每跑产物清单)+ §4.6.2(三类闸口表:Auto / Manual / Hard)+ §4.6.3(reviewer 7 条 checklist) | 满足 |
| 4 | **worked example**(audit F-18) | §4.9 (a)/(b)/(c)/(d) 四段:合成输入 / 命令 / YAML 输出 + header / 接续 I3 检查清单 | 满足 |

## 7. 下游影响

按 [`01-design-relationships.md`](../01-design-relationships.md) §4.9 / §4.6 出边:

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [B5 INS_Out_Bus 镜像同步流程](../B-contracts/B5-ins-bus-mirror.md) | Wave 5 sibling(per 关系图 §4.6 `B5 → F1`,B5 引用 I2 的 I/O 契约) | I2 §4.3.2 header comment block 形式 + §4.8 CLI / 退出码 — B5 据此锁定"抽取脚本"的对外 I/O 契约 |
| [I3 契约 diff 脚本设计](I3-contract-diff.md) | Wave 5 sibling(关系图 §4.9 `B1, B2, B3 ⇢ I2`;`B4 ↔ I3`;I3 消费 I2 的 YAML mirror) | I2 §4.3.2 YAML schema + §4.6.1 `mirror_report.json` 结构 — I3 输入面 |
| [B1 Bus 清单与 schema 设计](../B-contracts/B1-bus-inventory.md)(reviewed) | ⇢(I2 是 B1 的实现侧) | 镜像脚本承诺产出与 B1 字段元数据列等价的 YAML;B1 schema 修订后本文件 §4.3 / §4.7 jsonschema 来源同步 |
| [B2 Enum 清单与数值锁定](../B-contracts/B2-enum-inventory.md)(reviewed) | ⇢(I2 是 B2 的实现侧) | enum YAML 形态 + underlying int 处置规则 — B2 在 enum 表中声明 underlying 时,镜像脚本以 B2 为准 |
| [B3 Parameter schema 设计](../B-contracts/B3-parameter-schema.md)(reviewed) | ⇢(I2 是 B3 的实现侧) | PARAM/EXPORT YAML 形态;`classification: unknown` 占位由 B3 §4.2 分类方法 review-time 填充 |
| [I1 仓库初始化脚本设计](I1-init-script.md) | 可能受影响(MATLAB 侧 YAML → Simulink Bus helper 归属候选) | §4.3.1 / §4.7 占位 — Phase 2 实现阶段裁决 |
| [I4 Codegen 配置脚本设计](I4-codegen-config.md) | 可能受影响(同 I1) | 同上 |

注:本文件 `contract_impact: no` — 它是实现侧设计;触及 firmware 契约的设计在 B 区(B1/B2/B3/B5)登记。

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-07 | I2 author | 初稿,Wave 5。forward-cite B5 / I3(同 wave sibling,均 status: draft)。 |

## Self-check

- [x] frontmatter 完整,字段值合法
- [x] 上游文档全部存在且 status ≥ reviewed,**或**为本工作项所在 wave 的同 wave 兄弟(B5 / I3 forward-cite,§3 / §4.11 已注明 sibling 状态;非正式 co-seal,reviewer 须确认 wave-5 兄弟同时进入 reviewed)
- [x] 退出条件逐条复核完成,每条均给出依据(§6 四条:抽取规则 / 生成模板 / 人工 review 点 / worked example)
- [x] 引用路径全部可点击访问(架构 v1、A2、B1、B2、B3、B5、I3、I1、I4、RULES、00-design-plan、01-design-relationships)
- [x] 不存在 RULES §5 禁则中的内容(无 .slx 截图,伪代码标 `pseudo`,不复述 firmware 实现,不复述 bus/enum 字段定义,镜像产物记录 firmware commit hash 占位)
- [N/A] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记 — **本工作项 contract_impact=no**(I2 是 B1/B2/B3/B5 的实现侧,契约本身由 B 区拥有);故无需 INDEX 决策日志登记。
- [x] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径 — **I2 自身**不直接镜像契约,而是**产生**镜像产物;故 I2 的设计文档不必带 commit hash,但 I2 **产出的** YAML / sldd 必须带 `firmware_commit_sha` / `firmware_path` / `extracted_at`(per §4.3.2 / §4.9 header comment block;per RULES §5 + B5 §4.3 委派)。本设计已对此明确声明并在 §4.9 worked example 中演示。
- [x] 下游影响已沿关系图识别完毕(§7 列 B5 / I3 wave-5 兄弟 + B1/B2/B3 实现侧反向 ⇢ + I1/I4 helper 归属候选)
- [x] 文档不超出本工作项范围(无越权设计;不重定义 bus/enum/PARAM,不定义 diff 逻辑,不定义跨仓库 PR 流程 — 三者分别归属 B 区 / I3 / B5)
