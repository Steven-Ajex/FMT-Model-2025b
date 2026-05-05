---
work_item: A2
title: 命名与目录约定设计
upstream: ["架构 v1", "A1"]
contract_impact: no
status: reviewed
authored_at: 2026-05-05
last_reviewed_at: 2026-05-05
reviewer_verdict: pass
---

# A2 命名与目录约定设计

## 1. 目的

为 FMT-Model-2025b 全仓库锁定一套**统一的命名与目录约定**:文件、Simulink 库块、bus、enum、参数 struct 字段,以及单位、坐标系、角度的记法。本文件冻结 A1 §4.2 中归属 A2 的所有"命名/目录"类缺口,并为下游 A3、B 区(B1/B2/B3)、C/D/E/F/G/H/I 各区设计文档提供**唯一的命名规范来源**。

本文件**不**重定义 firmware 已固化的契约名(如 `FMS_init`、`FMS_PARAM`、`FMS_Out_Bus` 等);它只描述模型仓侧如何在新文件、新内部 bus、新内部 enum、新参数子项、新单位标签时遵循一致规则,并显式声明"何处必须与 firmware 已固化名严格对齐"。

## 2. 范围

**在范围:**

- 仓库目录结构在架构 v1 §6 基础上的命名细则与边界规则
- 文件命名(`.slx` / `.m` / `.md` / 数据文件)的命名风格与前缀
- Simulink 库块、模型块、子系统的命名规则
- Bus 命名规则(契约 bus vs 内部 bus 前缀策略;字段名风格)
- Enum 命名规则(类型名与成员名的大小写)
- 参数 struct 字段命名规则(模型仓侧新增字段)
- **单位约定**:SI 默认;非 SI 字段必须带显式后缀
- **坐标系约定**:NED 默认,body / vehicle / ENU 的后缀记法
- **角度单位约定**:rad 默认;deg 字段必须带显式后缀
- 与 firmware 已固化命名(`*_init` / `*_step` / `*_PARAM` / `*_EXPORT` / `*_Out_Bus` 等)的对齐规则

**不在范围(由其他工作项处理):**

- 具体 bus 字段清单与 schema(由 [B1 Bus 清单与 schema 设计](../B-contracts/B1-bus-inventory.md) 处理)
- 具体 enum 数值与字段清单(由 [B2 Enum 清单与数值锁定](../B-contracts/B2-enum-inventory.md) 处理)
- 具体参数字段清单与默认值(由 [B3 Parameter schema 设计](../B-contracts/B3-parameter-schema.md) 处理)
- 模块边界字段方向矩阵(由 [A3 模块边界信号清单细化](A3-module-boundaries.md) 处理)
- 跨速率边界处的 rate-transition 块命名(由 [A4 跨速率边界设计](A4-rate-boundaries.md) 处理)
- Variant Subsystem / Model Reference 的具体选型(由 [A5 变体策略设计](A5-variant-strategy.md) 处理)
- 共享库块的入选清单与归属(由 [A8 共享库块清单与归属](A8-shared-library-roster.md) 处理;本文件只给命名规则,不给清单)
- Codegen 配置项(由 [I4 Codegen 配置脚本设计](../I-tooling/I4-codegen-config.md) 处理)
- 测试用例文件命名(由 H 区设计项处理)

## 3. 依赖

| 上游 | 引用位置 | 用途 |
|---|---|---|
| 架构 v1 §5.1 firmware 契约 | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | 提取所有 firmware 已固化的命名(`*_init` / `*_step` / `*_PARAM` / `*_EXPORT` / `*_Out_Bus` / `model_info` / `period`),作为模型仓侧命名的硬约束 |
| 架构 v1 §6 推荐目录树 | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | 作为目录命名的基线;本文件在此基线上锁定细则与边界 |
| 架构 v1 §9.2.4 内部 bus 模块前缀示例 | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | 将其示例(`FMS_ModeMgr_Bus`、`CTRL_RateLoop_Bus`、`PLANT_AeroState_Bus`)上升为通用规则 |
| 架构 v1 §13 时序基线 | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | 作为时间单位与 period 元数据命名的依据 |
| 架构 v1 §14 参数/bus/enum 管理 | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | §14.2 mode-related enum 归属、§14.3 参数命名约束(`FMS_PARAM`/`CONTROL_PARAM`/`PLANT_PARAM`) |
| [A1 架构 v1 评审与缺口闭合](A1-v1-review.md) | A1 §4.2 / §4.4 | 提取归属 A2 的命名/目录类缺口并在本文件 §6 中逐条闭合 |
| RULES §1 / §4 / §5 | [链接](../RULES.md) | 设计文档自身的文件命名与引用规则(本文件遵守而非覆盖) |

注:A1 status 已为 reviewed(2026-05-05);架构 v1 是冻结基线。本文件不依赖任何同 batch 兄弟。

## 4. 设计内容

> 全文除 §4.10 单位/坐标系/角度三组约定外,所有规则均按"规则 → 反例 → 例外"组织。规则前缀 `R-`,反例前缀 `X-`,例外前缀 `E-`。

### 4.1 命名风格基线

**R-1.1 ASCII 唯一**:文件名、变量名、bus 名、enum 名、参数字段名一律使用 ASCII;不允许中文、空格、连字符以外的标点或非 ASCII 字符。

**R-1.2 大小写规则**(按对象类型):

| 对象 | 风格 | 例 |
|---|---|---|
| 仓库目录、子目录 | `lower_snake_case` | `model/shared/bus/`、`scripts/init/` |
| 文件名(`.slx` / `.m` / `.mat` / `.json`) | `lower_snake_case` | `fmt_model_init.m`、`mc_quad_x.slx` |
| 设计文档(`docs/design/<area>/`) | `<ID>-<kebab-case>.md` | `A2-naming-conventions.md` |
| Simulink 顶层模型与库块的**内部块名** | `PascalCase` 或 `Acronym_PascalCase` | `RateLoop`、`FMS_ModeMgr` |
| Bus 类型名 | `PascalCase_Bus` | `FMS_Out_Bus`、`PLANT_AeroState_Bus` |
| Enum 类型名 | `PascalCase`(无后缀) | `VehicleStatus`、`PilotMode` |
| Enum 成员名 | `UPPER_SNAKE_CASE` | `STATUS_ARM`、`MODE_MANUAL` |
| 参数 struct 名 | `UPPER_SNAKE_CASE` | `FMS_PARAM`、`CONTROL_PARAM` |
| 参数 struct 字段名 | `lower_snake_case` | `att_p_gain`、`vel_xy_lim_mps` |
| MATLAB 函数 / 脚本符号 | `lower_snake_case` | `fmt_model_init`、`mirror_ins_bus` |

**R-1.3 长度上限**:任一标识符 ≤ 63 字符(MATLAB 标识符上限),且建议 ≤ 40 字符以利可读性与代码生成后的 C 名长度。

**X-1.1**(反例)`FMS-Out-Bus`(连字符)、`fmsoutbus`(无可读分隔)、`FMSOutBus`(违反 `_Bus` 后缀规则)、`fmsOutBus`(camelCase 不属于任何类别)。

### 4.2 目录契约(在架构 v1 §6 基础上的细则)

**R-2.1 目录树以架构 v1 §6 为唯一基线**;本文件不重复其结构,仅锁定其中**未在 v1 中明确**的命名细则。

**R-2.2 vehicle 目录命名**:在 `model/<module>/vehicles/<vehicle>/` 下,`<vehicle>` 仅允许下列受控集合:`multicopter` / `fixwing` / `vtol` / `boat` / `car` / `submarine` / `template`。新增机型须先在本文件 §4.2 追加并通过评审。

**R-2.3 export 子树命名**:`export/firmware/<module>/<vehicle>/`(闭合 A1 §4.2 §11.3)。`<module>` 取值受控为 `plant` / `fms` / `controller`(无 `ins/`,因 INS 由 firmware 拥有)。`<vehicle>` 取值同 R-2.2。

**R-2.4 share 与 leaf 边界目录**:每个 `model/<module>/` 下必须严格分为 `shared/` 与 `vehicles/` 两个子目录;不允许在 `model/<module>/` 直挂 `.slx`(避免出现"既非 shared 也非 leaf"的歧义层)。

**R-2.5 子产物子目录**:设计文档的子产物(图、表、伪代码片段)放在与主文档同名的子目录,即 `docs/design/<area>/<id>-<slug>/`(此为 RULES §1 既有规则的复述与冻结)。

**R-2.6 占位目录策略**:对暂未交付的 vehicle 目录(fixwing/vtol/boat/car/submarine/template),在设计阶段**不**预先创建空目录;由各 leaf 工作项首次需要时按上述 R-2.2 命名创建。例外:模板目录 `template/` 由 A5 决定是否在 Phase 1 创建(本文件不裁决,仅给名)。

**X-2.1**(反例)`model/multicopter/fms/`(模块/机型层级倒置)、`model/fms/multicopter.slx`(直挂在模块根)、`export/firmware/multicopter/fms/`(模块/机型层级倒置)。

### 4.3 文件命名

**R-3.1 顶层 Simulink 模型文件**:`<module>_<vehicle>.slx`,例 `fms_multicopter.slx`、`controller_multicopter.slx`、`plant_multicopter.slx`。位置在 `model/<module>/vehicles/<vehicle>/` 下。

**R-3.2 Simulink 共享库文件**:`<scope>_lib.slx`,`<scope>` 取自模块前缀或共享层短名,例 `fms_shared_lib.slx`、`ctrl_shared_lib.slx`、`plant_shared_lib.slx`、`core_lib.slx`(对应 `model/shared/lib/`)。

**R-3.3 数据字典文件**(若 B1 选择 SLDD 形态):`<scope>.sldd`,例 `fmt_buses.sldd`、`fmt_enums.sldd`。

**R-3.4 Bus/enum 定义脚本**(若 B1 选择脚本形态):`define_<scope>_bus.m` / `define_<scope>_enum.m`,例 `define_fms_out_bus.m`。

**R-3.5 参数默认值文件**:`<vehicle>_<module>_param.m`,例 `multicopter_fms_param.m`。位置在 `model/shared/params/defaults/` 或与 vehicle leaf 同目录(B3 决定)。

**R-3.6 脚本文件**:`<verb>_<object>.m`,动词优先,例 `mirror_ins_bus.m`、`build_multicopter.m`、`run_smoke.m`。例外:仓库唯一入口 `FMT_Model_Init.m` 保留 PascalCase(沿袭 FMT 历史命名,作为可见入口);I1 设计在其退出条件中显式登记此例外。

**R-3.7 设计文档**:`<ID>-<kebab-case>.md`(RULES §1 既定),不在本文件重复约束。

**X-3.1**(反例)`FMS_Multicopter.slx`(违反 R-1.2 文件 lower_snake_case)、`multicopter_fms.slx`(违反 R-3.1 模块在前)、`init.m`(无主语)、`build.m`(无对象)。

### 4.4 Simulink 块命名

**R-4.1 子系统块名**:`PascalCase`,语义化名词或动名词;例 `RateLoop`、`AttitudeLoop`、`ModeManager`、`CommandShaper`、`SafetyMonitor`、`OutputAssembler`。

**R-4.2 库块名前缀**:模型仓共享库内的可复用块在块名前加**模块前缀**,与内部 bus 前缀一致(见 R-5.2):`FMS_*` / `CTRL_*` / `PLANT_*` / `CORE_*`。例 `CTRL_RateLoopShell`、`CORE_RateLimiter`、`CORE_LowPass1`。

**R-4.3 顶层模型根名**:与 R-3.1 文件名(去 `.slx`)一致;Simulink 强约束。

**R-4.4 Sample Time / Rate transition 块**:命名前缀 `RT_`,后缀注明源→目速率,例 `RT_1ms_to_5ms`、`RT_5ms_to_20ms`(供 A4 / G2 在调度图中复用;本文件只给前缀,不给清单)。

**X-4.1**(反例)`Subsystem1`(默认名)、`my_loop`(snake_case 用错对象)、`fmsModeMgr`(camelCase)。

### 4.5 Bus 命名

**R-5.1 契约 bus**:沿袭 firmware 已固化名,**不得**改名,**不得**自创别名。受控清单(B1 落到字段):
- `FMS_Out_Bus`(FMS 输出 → Controller 输入)
- `Control_Out_Bus`(Controller 输出 → Plant 输入,FMS 可选回送)
- `Plant_States_Bus`、`Extended_States_Bus`(Plant 输出)
- `INS_Out_Bus`(INS → FMS / Controller,**firmware 拥有,镜像入仓**)
- `Pilot_Cmd_Bus` / `GCS_Cmd_Bus` / `Auto_Cmd_Bus` / `Mission_Data_Bus`(FMS 输入)
- `Environment_Info_Bus` / `States_Init_Bus`(Plant 输入)
- `IMU_Out_Bus` / `MAG_Out_Bus` / `Barometer_Out_Bus` / `GPS_Out_Bus` / `Airspeed_Out_Bus`(Plant 模拟传感器输出;字段以 firmware 为准)

**R-5.2 内部 bus 前缀**:任一**未跨模块**的 bus 类型名必须带模块前缀,以便与契约 bus 视觉区隔:
- `FMS_*_Bus`(FMS 内部),例 `FMS_ModeMgr_Bus`、`FMS_Safety_Bus`、`FMS_ShaperOut_Bus`
- `CTRL_*_Bus`(Controller 内部),例 `CTRL_RateLoop_Bus`、`CTRL_PosErr_Bus`
- `PLANT_*_Bus`(Plant 内部),例 `PLANT_AeroState_Bus`、`PLANT_RotorState_Bus`
- `CORE_*_Bus`(`model/shared/lib/` 横切共享),少用;若需跨多模块复用结构请走 B 区评审。

**R-5.3 Bus 字段命名**:`lower_snake_case`;字段名包含**单位**与**坐标系**后缀(见 §4.10),例 `vel_ned_mps`、`ang_rate_b_radps`、`yaw_rad`、`alt_m`。

**R-5.4 验证字段命名**:与字段对应的有效性/质量字段统一以 `_valid`(布尔)、`_quality`(0..1)或 `_status`(enum)结尾;例 `gps_valid`、`ins_quality`、`bat_status`。具体字段由 B1 决定,本文件只锁后缀风格。

**E-5.1 例外**:firmware `*_types.h` 中已固化的字段名优先;本文件命名规则**仅作用于模型仓侧新增字段**。`B5 INS_Out_Bus` 镜像产物完全跟随 firmware 字段名,无论是否符合 R-5.3 后缀规则。

**X-5.1**(反例)`fms_out_bus`(契约 bus 应保留 `FMS_Out_Bus`)、`mode_mgr_bus`(无模块前缀)、`Velocity`(缺单位/坐标系后缀)。

### 4.6 Enum 命名

**R-6.1 Enum 类型名**:`PascalCase`,语义化名词,**无**后缀(无 `_E` 也无 `_Type`)。例 `VehicleStatus`、`VehicleState`、`PilotMode`、`CtrlMode`、`ErrorCode`。

**R-6.2 Enum 成员名**:`UPPER_SNAKE_CASE`,且加**类型短前缀**避免成员名跨 enum 冲突;前缀短形参考:
- `VehicleStatus` 成员前缀 `STATUS_`,例 `STATUS_DISARM`、`STATUS_ARM`
- `VehicleState` 成员前缀 `STATE_`
- `PilotMode` 成员前缀 `PMODE_`
- `CtrlMode` 成员前缀 `CMODE_`
- `ErrorCode` 成员前缀 `ERR_`

**R-6.3 数值锁定**:任何已固化于 firmware 的 enum 数值,在模型仓侧不得重新分配数值;本文件不列出具体数值(由 [B2 Enum 清单与数值锁定](../B-contracts/B2-enum-inventory.md) 处理)。

**R-6.4 命名归属**:架构 v1 §14.2 规定 mode-related enum 由 FMS shared 拥有;本文件命名规则不再分散到 vehicle leaf,所有契约 enum 一律置于 `model/shared/enum/`。

**E-6.1 例外**:firmware 已固化的成员名(例如某些大写带数字成员)按现状保留;A2 规则只作用于**新增**成员命名。具体差异由 B2 列出。

**X-6.1**(反例)`vehicleStatus`(类型名 camelCase)、`Status_Arm`(成员混合大小写)、`ARM`(无类型短前缀,跨 enum 易冲突)。

### 4.7 参数 struct 命名

**R-7.1 顶层参数 struct 名固化**:`FMS_PARAM` / `CONTROL_PARAM` / `PLANT_PARAM`(架构 v1 §5.1.2、§14.3),不得改名。

**R-7.2 EXPORT struct 名固化**:`FMS_EXPORT` / `CONTROL_EXPORT` / `PLANT_EXPORT`(架构 v1 §5.1.2),不得改名。本文件不列出其字段(由 [B3](../B-contracts/B3-parameter-schema.md) 处理),只锁定其字段命名风格 R-7.4。

**R-7.3 子结构与字段名风格**:`PARAM` / `EXPORT` 内的字段名一律 `lower_snake_case`;若使用嵌套 struct,嵌套字段名仍 `lower_snake_case`,嵌套类型(若需要单独命名)采用 `<MODULE>_<Sub>_PARAM` 形式,例 `CONTROL_RateLoop_PARAM`(需 B3 确认是否使用)。

**R-7.4 字段单位与坐标系后缀**:参数字段的**最末一段**必须包含单位/坐标系后缀(见 §4.10)。例:
- `vel_xy_lim_mps`(水平速度上限,m/s)
- `att_p_gain_radps_per_rad`(姿态比例增益,rad/s 每 rad)
- `yaw_rate_lim_radps`(偏航率上限,rad/s)
- `geofence_radius_m`(电子围栏半径,m)

**R-7.5 runtime-tunable vs compile-inlined**:命名风格不区分;分类记录由 [B3](../B-contracts/B3-parameter-schema.md) 在每字段元数据中标注。本文件**不**给字段起按分类前缀(避免改一次分类要重命名)。

**X-7.1**(反例)`FmsParam`(违反 R-7.1)、`AttPGain`(PascalCase 用错对象,且无单位)、`vel_xy_lim`(缺单位后缀)。

### 4.8 与 firmware 已固化命名的对齐表

下表把 firmware 已固化的、模型仓**必须照搬**的名字集中列出;**任何下游设计文档涉及这些名字时一律以本表为准**:

| 类别 | firmware 已固化名 | 来源(架构 v1) | 模型仓侧规则 |
|---|---|---|---|
| 模块集成入口符号 | `plant_interface_init` / `plant_interface_step` | §5.1.1 | 模型仓不直接命名(由 firmware 集成层产出),但生成代码必须能被其调用 |
| 模块集成入口符号 | `fms_interface_init` / `fms_interface_step` | §5.1.1 | 同上 |
| 模块集成入口符号 | `control_interface_init` / `control_interface_step` | §5.1.1 | 同上 |
| 模块集成入口符号 | `ins_interface_init` / `ins_interface_step` | §5.1.1 | **firmware 拥有**;模型仓不生成 |
| 生成模型入口符号 | `Plant_init` / `Plant_step` | §5.1.2 | 必须由模型仓 ert.tlc 生成产物提供;Simulink 顶层模型根名应导致 codegen 产出此符号 → 由此推出 R-3.1 文件名规则 |
| 生成模型入口符号 | `FMS_init` / `FMS_step` | §5.1.2 | 同上 |
| 生成模型入口符号 | `Controller_init` / `Controller_step` | §5.1.2 | 同上 |
| EXPORT struct | `PLANT_EXPORT` / `FMS_EXPORT` / `CONTROL_EXPORT` | §5.1.2 | 名固化,字段由 B3 |
| PARAM struct | `PLANT_PARAM` / `FMS_PARAM` / `CONTROL_PARAM` | §5.1.2 / §14.3 | 名固化,字段由 B3 |
| `model_info` 元数据 | `model_info[]`、`period` | §5.1.2 | 字段名固化;字段值由 I4 codegen 配置;本文件锁名 |
| 契约 bus | 见 R-5.1 清单 | §5.1.4 / §5.1.5 / §5.1.6 / §5.1.7 / §5.1.8 | 名固化;字段由 B1 |
| 契约 enum | mode/state/error 系列 | §5.1.9 / §14.2 | 名与数值固化;本文件给命名风格,数值由 B2 |

**R-8.1 顶层模型根名推导**:Simulink ert.tlc 默认会把根模型名作为生成函数名前缀。为使生成代码符合 R-1.2 的 `Plant_init` / `FMS_init` / `Controller_init`,顶层 `.slx` 的**模型根名**必须是 `Plant` / `FMS` / `Controller`(注意大小写),**而文件名**仍按 R-3.1 走 `<module>_<vehicle>.slx`(Simulink 允许文件名与根模型名不同)。**这是模型仓侧 PascalCase 模型根名 vs 文件名 lower_snake_case 的硬接缝**;I4 codegen 配置必须显式校验此映射。

### 4.9 内部 bus / 内部 enum / 内部 param 的"不溢出"边界

**R-9.1 内部 bus / enum / param 不得出现在 firmware-visible 接口**:即不得作为 `FMS_Out_Bus` / `Control_Out_Bus` / `Plant_States_Bus` 等契约 bus 的字段类型,也不得出现在 `*_PARAM` / `*_EXPORT` 的字段类型(仅以基础类型与契约 bus / 契约 enum 暴露)。

**R-9.2 区分手段**:本文件 R-5.2 / R-6.1 给出的命名规则即为视觉区分(模块前缀 vs 无前缀);[B4 契约 diff](../B-contracts/B4-contract-diff.md) 在脚本侧负责检测违反。

### 4.10 单位、坐标系、角度三组约定

#### 4.10.1 单位约定(SI 默认)

**R-10.1 默认 SI**:所有物理量字段默认使用 SI 基本单位或一阶导出单位。具体清单(下游 B1/B3 必须遵守):

| 量 | 默认 SI 单位 | 字段后缀 |
|---|---|---|
| 长度 | m | `_m` |
| 速度 | m/s | `_mps` |
| 加速度 | m/s² | `_mps2` |
| 力 | N | `_n` |
| 力矩 | N·m | `_nm` |
| 质量 | kg | `_kg` |
| 时间 | s | `_s` |
| 角度 | rad | `_rad`(详见 §4.10.3) |
| 角速度 | rad/s | `_radps` |
| 角加速度 | rad/s² | `_radps2` |
| 频率 | Hz | `_hz` |
| 电压 | V | `_v` |
| 电流 | A | `_a` |
| 温度 | K | `_k` |
| 压力 | Pa | `_pa` |
| 百分比/归一化 | 无量纲 0..1 | `_n01`(归一化)或 `_pct`(百分比 0..100) |

**R-10.2 非 SI 字段必须显式后缀**:任何因接口或行业惯例必须使用非 SI 单位的字段,字段名末段**必须**写入实际单位;例:
- `temp_c`(摄氏度,非 K)
- `alt_ft`(英尺,非 m)
- `speed_kt`(节,非 m/s)
- `period_ms`(毫秒,非 s — 注:`period` 是 firmware EXPORT struct 字段,固化为 ms,见 §4.8)
- `bat_pct`(电量百分比 0..100)

**R-10.3 时间戳与周期**:时间戳字段统一 `_us`(微秒;具体由 [A7 时间与时间戳约定](A7-time-conventions.md) 锁定 epoch 与回卷规则);周期字段统一 `_ms`(与 firmware `period` 字段单位一致)。本文件仅锁后缀,数值与 epoch 由 A7。

**X-10.1**(反例)`velocity`(无单位)、`alt_height`(冗余且无单位)、`period`(无单位 — 但因是 firmware 已固化字段名,作为 §4.8 例外保留)。

#### 4.10.2 坐标系约定(NED 默认)

**R-10.4 默认坐标系 = NED(North-East-Down,惯性/世界系)**:任何位置、速度、加速度,只要不显式标注,即视为 NED。

**R-10.5 坐标系后缀**:字段名中必须以下列后缀之一表明坐标系(放在物理量后、单位后缀之前):

| 坐标系 | 后缀 | 含义 |
|---|---|---|
| NED 惯性系(默认) | `_ned` | North/East/Down,世界固连 |
| 机体系 | `_b` | Body,与机体姿态同动 |
| 机载平台/载具系 | `_v` | Vehicle frame(若与 body 不同,例 VTOL 倾转) |
| ENU(可选,仅在与外部 ENU 接口对接时用) | `_enu` | East/North/Up |
| FRD(机体 forward/right/down,通常与 body 等价) | `_frd` | 仅当与 `_b` 含义不一致时使用 |

完整字段命名顺序为:`<量名>_<坐标系>_<单位>`,例 `vel_ned_mps`、`acc_b_mps2`、`pos_ned_m`、`ang_rate_b_radps`。

**R-10.6 不需坐标系标注的量**:标量物理量(温度、电压、电池剩余、高度 above-ground 等)无坐标系后缀;角度作为标量姿态分量(`yaw_rad` / `pitch_rad` / `roll_rad`)默认 NED→body 的 ZYX 欧拉角,不再加 `_ned` 后缀,但需在 B1 字段元数据中显式记录欧拉角顺序与基坐标系。

**E-10.1 firmware 已固化字段例外**:若某 firmware 字段名不带坐标系后缀(例如某些历史字段直接叫 `velocity`),按 §4.8 沿用原名;模型仓侧新增字段必须满足 R-10.5。

**X-10.2**(反例)`vel_mps`(缺坐标系)、`pos_b_m`(机体系位置无意义)、`accel_world`(口语坐标系名)。

#### 4.10.3 角度单位约定(rad 默认)

**R-10.7 默认 rad**:所有角度、角速度、角加速度字段默认弧度制。

**R-10.8 deg 例外必须显式**:任何使用度的字段必须带 `_deg` / `_degps` / `_degps2` 后缀;典型场景:
- 与 GCS / mission planner 的人机界面字段(航向显示、航点航向)
- 经纬度(`lat_deg`、`lon_deg`)
- 历史 firmware 字段(若存在)

**R-10.9 经纬度专项**:`lat_deg` / `lon_deg`(度)+ `alt_m`(米)或 `alt_ft`(英尺)是 mission/home/wp 类字段的标准三元组;具体字段顺序与类型由 B1。

**R-10.10 内部计算**:模块内部 bus(`FMS_*_Bus` / `CTRL_*_Bus`)中的角度量**强制** rad,以避免 deg/rad 在级联中混用导致量纲错误。deg 仅出现在面向 GCS/外设的边界。

**X-10.3**(反例)`heading`(缺单位)、`yaw_angle_deg` 用于 Controller 内部 bus(违反 R-10.10)。

### 4.11 与下游工作项的命名接缝

下表显式声明本文件之外**何处由谁负责命名细节**,作为下游作者的入口指引:

| 接缝 | 本文件给出 | 下游负责 |
|---|---|---|
| 契约 bus 字段名 | 模块前缀、单位/坐标系/角度后缀风格 | [B1](../B-contracts/B1-bus-inventory.md) |
| 契约 enum 成员名 | 类型/成员大小写、类型短前缀 | [B2](../B-contracts/B2-enum-inventory.md) |
| `*_PARAM` 字段名 | snake_case + 单位后缀 | [B3](../B-contracts/B3-parameter-schema.md) |
| 模块内部 bus 前缀 | `FMS_*` / `CTRL_*` / `PLANT_*` / `CORE_*` | [C2](../C-plant/C2-plant-structural.md) / [D2](../D-fms/D2-fms-structural.md) / [E2](../E-controller/E2-controller-structural.md) |
| 共享库块入选与归属 | 块名前缀(R-4.2) | [A8](A8-shared-library-roster.md) |
| Rate transition 块名 | `RT_<src>_to_<dst>` | [A4](A4-rate-boundaries.md) / [G2](../G-harness/G2-rate-scheduling.md) |
| 字段方向矩阵 | — | [A3](A3-module-boundaries.md) |
| Variant 块/Model Reference 命名 | 沿 R-1.2 PascalCase | [A5](A5-variant-strategy.md) |
| Codegen 配置中的根模型名校验 | R-8.1 推导规则 | [I4](../I-tooling/I4-codegen-config.md) |
| 镜像脚本输出文件名 | R-3.4 / R-3.6 | [I2](../I-tooling/I2-bus-enum-mirror.md) |

## 5. 已知风险与悬而未决问题

- **风险:R-8.1 模型根名 ≠ 文件名带来的 codegen 误配置**
  - 影响:I4。若 ert.tlc 配置或 init 脚本错把文件名当模型名,生成代码会出现 `fms_multicopter_init` 而非 `FMS_init`,直接破坏 firmware 集成。
  - 处置:I4 设计中显式增加根模型名校验测试用例;B4 契约 diff 把"`FMS_init`/`Controller_init`/`Plant_init` 必须存在于生成产物"作为强约束(已在 00-design-plan §4.B 的 B4 退出条件 (e) 项中列入)。

- **风险:firmware 已固化字段不符合本文件 §4.10 后缀规则,易引导新作者照抄旧风格**
  - 影响:B1 / B3。若 B1 列字段时未在文档中注明哪些是"firmware 沿袭名"(违反 R-5.3 / R-7.4 但不可改),新作者可能在内部 bus / 新参数中也省略后缀。
  - 处置:B1 / B3 字段表必须含"unit suffix compliant: yes / firmware-legacy"列;B4 契约 diff 在 firmware-legacy 字段上仅做"字段顺序+类型"检查,不强求改名。

- **风险:"内部 bus 模块前缀" 与 "shared/lib 共享块前缀 `CORE_`" 之间的横切灰区**
  - 影响:A8、C2/D2/E2。若一个块从单模块复用上升到跨模块共享,需要重命名 `FMS_X` → `CORE_X`,触发跨文档变更。
  - 处置:A8 设计共享库块入选清单时,对每个候选块标注"提升到 CORE_ 的代价 = 重命名 + 触及调用者";本文件不锁清单,由 A8 治理。

- **(open)** A1 §4.4 曾提出 *(proposed)* A6 = 共享库块清单与归属总表,可选择并入 A2。**处置**:00-design-plan §4.A 已将 A6 重新定义为"Init/Reset 状态契约"、A8 为"共享库块清单与归属",且 A1 提出的 *(proposed)* A6 议题已由 00-design-plan 的 A8 承担。本文件不重复 A8 的清单职责,只在 §4.4 R-4.2 / §4.11 中给"共享库块前缀命名规则",将清单留给 A8。此处理方式已与 A1 §4.4 的"合并到 A2"备选方案一致(命名义务在 A2,清单义务在 A8)。

- **(open)** 单位后缀风格选取(`_mps` vs `_m_s`,`_radps` vs `_rad_s`)的可读性争议
  - 影响:B1 / B3 字段命名长度。
  - 处置:本文件选 `_mps` / `_radps`(无下划线,贴近 PX4/Ardupilot 习惯且更短),冻结于本版本;后续若发现可读性问题,走 RULES §10 变更日志整体迁移。

## 6. 退出条件复核

[`00-design-plan.md §4.A`](../00-design-plan.md) 中 A2 的退出条件原文:

> 文件名、库块名、bus 名、enum 名、参数 struct 字段名的命名规则全部定稿;**单位与坐标系约定(SI 单位、NED/ENU/body 等坐标系记法、角度 rad vs deg)写入命名规则**

逐条复核:

| # | 退出条件分解 | 本文档依据 | 状态 |
|---|---|---|---|
| 1 | 文件名命名规则 | §4.3(R-3.1 ~ R-3.7) | 满足 |
| 2 | (Simulink)库块名命名规则 | §4.4(R-4.1 ~ R-4.4) + §4.11(共享库块前缀对接 A8) | 满足 |
| 3 | Bus 名命名规则 | §4.5(R-5.1 ~ R-5.4)+ §4.9(内部/契约边界) | 满足(字段清单按 §2 不在范围,留 B1) |
| 4 | Enum 名命名规则 | §4.6(R-6.1 ~ R-6.4) | 满足(数值清单留 B2) |
| 5 | 参数 struct 字段名命名规则 | §4.7(R-7.1 ~ R-7.5) | 满足(字段清单留 B3) |
| 6 | 单位约定(SI 默认) + 非 SI 显式后缀 | §4.10.1(R-10.1 ~ R-10.3) | 满足 |
| 7 | 坐标系约定(NED 默认) + 后缀记法 | §4.10.2(R-10.4 ~ R-10.6) | 满足 |
| 8 | 角度单位约定(rad 默认) + deg 例外 | §4.10.3(R-10.7 ~ R-10.10) | 满足 |

附:对 [A1 §4.2 / §4.3 / §4.4](A1-v1-review.md) 中归属 A2 的缺口逐条闭合:

| 来源(A1) | 缺口简述 | 本文档闭合位置 |
|---|---|---|
| A1 §4.2 §2.6 | "Normalize bus, enum, parameter, and codegen management" 命名规范 | §4.5(bus) + §4.6(enum) + §4.7(param) + §4.8(对齐表)|
| A1 §4.2 §5.2.1 | 一份 source of truth 的目录与命名形态 | §4.2 + §4.3(目录与文件命名);来源形态(SLDD vs script)由 B1 决策但本文件已给两种形态的文件名规则(R-3.3 / R-3.4) |
| A1 §4.2 §5.2.6 | "Standardize directory names, harness placement, and export staging" 命名 | §4.2(R-2.1 ~ R-2.6),harness 顶层布局留 G1 |
| A1 §4.2 §11.3 | `export/firmware/<module>/<vehicle>/` 命名 | §4.2 R-2.3 |
| A1 §4.2 §9.2.4 | 内部 bus 模块前缀规则 | §4.5 R-5.2 |
| A1 §4.4 *(proposed A6)* "共享库块命名 + 横向 review 检查表" 议题 | 命名规则 | §4.4 R-4.2 + §4.11(清单义务交给 A8) |

退出条件全部满足;状态可由 reviewer 升级到 reviewed。

## 7. 下游影响

按 [`01-design-relationships.md`](../01-design-relationships.md) 出边:

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| [A3 模块边界信号清单细化](A3-module-boundaries.md) | → | 字段命名风格(§4.5 R-5.3、§4.10 单位/坐标系/角度三组后缀);A3 列字段时直接引用本文件后缀表 |
| [B1 Bus 清单与 schema 设计](../B-contracts/B1-bus-inventory.md) | ⇢ | bus 类型名前缀(R-5.1 / R-5.2)、字段命名风格、firmware-legacy 字段豁免规则、字段元数据所需"unit suffix compliant"列 |
| [B2 Enum 清单与数值锁定](../B-contracts/B2-enum-inventory.md) | ⇢ | enum 类型名/成员名风格(R-6.1 / R-6.2)、归属规则(R-6.4) |
| [B3 Parameter schema 设计](../B-contracts/B3-parameter-schema.md) | ⇢ | `*_PARAM` / `*_EXPORT` 名固化(R-7.1 / R-7.2)、字段命名风格(R-7.3 / R-7.4)、单位后缀义务 |
| [B4 契约 diff 策略设计](../B-contracts/B4-contract-diff.md) | ⇢ | 契约名清单(§4.8)、firmware-legacy 字段处理规则(§5 风险 2 处置) |
| [B5 INS_Out_Bus 镜像同步流程](../B-contracts/B5-ins-bus-mirror.md) | ⇢ | INS_Out_Bus 镜像产物的文件命名(R-3.4)、firmware 字段名优先规则(E-5.1) |
| C 区(C1/C2/C3/C4) | ⇢ | 内部 bus 前缀 `PLANT_*`、Plant 文件命名(R-3.1)、共享库前缀 `CORE_` |
| D 区(D1/D2/D3/D4/D5/D6) | ⇢ | 内部 bus 前缀 `FMS_*`、FMS 文件命名、Mode/State 命名归属(R-6.4) |
| E 区(E1/E2/E3/E4/E5) | ⇢ | 内部 bus 前缀 `CTRL_*`、Controller 文件命名、共享 cascade 块前缀(R-4.2) |
| F 区(F1/F2/F3) | ⇢ | INS_Out_Bus 沿 firmware 字段名(E-5.1);ins_stub 文件位于 `model/harness/ins_stub/` |
| [G1 MIL 顶层结构设计](../G-harness/G1-mil-toplevel.md) | ⇢ | harness 子目录命名(R-2.1 沿架构 v1 §6)、顶层模型文件名 |
| [G2 多速率调度设计](../G-harness/G2-rate-scheduling.md) | ⇢ | rate transition 块前缀 `RT_`(R-4.4) |
| [G3 日志与可观测性设计](../G-harness/G3-logging.md) | ⇢ | logsout 信号名风格(沿 §4.5 R-5.3) |
| [H 区(H1/H2/H3/H4)](../H-verification/) | ⇢ | 场景与基线文件命名(R-3.6 verb_object 风格)、指标字段名沿用 §4.10 单位后缀 |
| [I1 仓库初始化脚本设计](../I-tooling/I1-init-script.md) | ⇢ | `FMT_Model_Init.m` 例外(R-3.6 E-3 等价说明)、init 输出 base workspace 中变量命名沿 §4.5 |
| [I2 Bus/enum 镜像脚本设计](../I-tooling/I2-bus-enum-mirror.md) | ⇢ | 输出文件命名(R-3.4)、镜像产物保留 firmware 字段名(E-5.1) |
| [I3 契约 diff 脚本设计](../I-tooling/I3-contract-diff.md) | ⇢ | §4.8 对齐表作为 diff 范围之一(符号存在性) |
| [I4 Codegen 配置脚本设计](../I-tooling/I4-codegen-config.md) | ⇢ | R-8.1 根模型名 vs 文件名硬接缝校验 |
| [I5 仿真运行/批量回归脚本设计](../I-tooling/I5-sim-runner.md) | ⇢ | 脚本动名词命名(R-3.6) |

整体而言,本文件是 A 区之后**几乎所有**设计文档的横切引用源;评审通过后任何对本文件的修订都需沿全部出边对受影响下游加 `Affected by upstream change` 标记(RULES §10)。

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| 2026-05-05 | A2 author | 初稿 |

## Self-check

- [x] frontmatter 完整,字段值合法
- [x] 上游文档全部存在且 status ≥ reviewed(架构 v1 已发布并锚定 2026-05-05;A1 status=reviewed)
- [x] 退出条件逐条复核完成,每条均给出依据
- [x] 引用路径全部可点击访问
- [x] 不存在 RULES §5 禁则中的内容(无 .slx 截图占位、无可执行 .m 代码、无 bus/enum 字段表、无 PR/branch 名)
- [x] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记(本文件 contract_impact=no — 命名规则不改 firmware 字段顺序、enum 数值或符号名,仅约束模型仓侧新增标识符;§4.8 对齐表为引用,非定义,N/A)
- [x] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径(本文件不镜像具体契约字段,仅引用 firmware 已固化的命名约束;commit hash 由 B1/B2/B3/B5 在镜像产物中记录,N/A)
- [x] 下游影响已沿关系图识别完毕(§7 覆盖 A3/B 全/C/D/E/F/G/H/I 全)
- [x] 文档不超出本工作项范围(无越权设计 — 字段清单/数值/方向/选型/清单全部留给指定下游)
