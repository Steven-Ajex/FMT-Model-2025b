# FMT-Model-2025b 设计文档规则

最后更新:2026-05-05
适用范围:`docs/design/` 下所有设计文档与评审

## 1. 文件命名与位置

- 每个工作项**一个**主文档:`<id>-<slug>.md`
  - 例:`A1-v1-review.md`、`B1-bus-inventory.md`、`D3-mode-manager.md`
- 文档存放路径:`docs/design/<letter>-<area>/<id>-<slug>.md`
- 子产物(图、表、伪代码片段)放在同目录 `<id>-<slug>/` 子文件夹内
- 模板:[`_templates/design-doc-template.md`](_templates/design-doc-template.md)

## 2. 必备 Frontmatter

每个设计文档**首行起**必须包含:

```yaml
---
work_item: <ID>                      # 例如 A1
title: <短标题>
upstream: [<ID>, <ID>, ...]          # 上游工作项 ID 或 "架构v1"
contract_impact: <yes|no>            # 是否触及 firmware 可见契约
status: <draft|reviewed|accepted>    # 见 §7 状态机
authored_at: <YYYY-MM-DD>
last_reviewed_at: <YYYY-MM-DD>       # 无评审时为空
reviewer_verdict: <pass|changes-requested|none>
---
```

## 3. 必备章节(按顺序)

1. **目的** — 一句话:本文件解决什么设计问题
2. **范围** — 显式列出"在范围"和"不在范围"
3. **依赖** — 上游设计文档/契约/架构章节,带链接
4. **设计内容** — 主体;子结构由本工作项的退出条件决定
5. **已知风险与悬而未决问题** — 项目符号列表
6. **退出条件复核** — 逐条对照 [`00-design-plan.md`](00-design-plan.md) 该工作项的退出条件
7. **下游影响** — 按 [`01-design-relationships.md`](01-design-relationships.md) 出边列出受影响下游
8. **Self-check** — §6 的复选清单

## 4. 引用规则

- 同 design 目录:`[<ID> <短标题>](../<area>/<id>-<slug>.md)`
- 架构 v1:`[架构 v1 §N](../../architecture/2026-05-05-fmt-model-architecture-v1.md)`
- firmware 头文件路径用相对路径或注明 `FMT-Firmware/src/...`
- **禁止重复定义**:bus/enum/参数一旦在 B 区定稿,其他文档只引用,不复述
- 引用 firmware 接口时,引用的字段必须实际存在于 `*_types.h`

## 5. 内容禁则

| 禁则 | 替代做法 |
|---|---|
| `.slx` 截图占位 | 文本/Mermaid 描述子系统拓扑 |
| 可执行 `.m` 代码 | 标 `pseudo` 的伪代码块 |
| 重复 firmware 实现细节 | 只描述模型仓侧 |
| PR/branch 名等易过期信息 | 用决策日期 + 决策日志锚点 |
| 多文档重复 bus/enum 字段表 | 在 B 区定义,他处只引用 |
| 未注明依赖的"凭空"设计 | 必须列在 §3 依赖章节 |
| 镜像自 firmware 的契约不记录来源版本 | 在 §3 依赖中记录 `FMT-Firmware` 的 commit hash + 文件相对路径 |

## 6. Self-check 清单(文末必带,原样复制)

```markdown
## Self-check

- [ ] frontmatter 完整,字段值合法
- [ ] 上游文档全部存在且 status ≥ reviewed,**或**为本工作项所在 co-seal batch 的同批兄弟(允许同批互相引用,需在 §3 注明 batch 名)
- [ ] 退出条件逐条复核完成,每条均给出依据
- [ ] 引用路径全部可点击访问
- [ ] 不存在 §5 禁则中的内容
- [ ] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记
- [ ] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径
- [ ] 下游影响已沿关系图识别完毕
- [ ] 文档不超出本工作项范围(无越权设计)
```

## 7. 文档状态机

| 状态 | 含义 | 进入条件 |
|---|---|---|
| `draft` | 作者撰写中或已提交但未评审 | 文件创建即此状态 |
| `reviewed` | 评审通过(reviewer_verdict=pass) | `fmt-design-review` 出具 pass |
| `accepted` | 用户最终确认 | 用户显式接受 |
| `changes-requested` | 评审打回 | reviewer_verdict=changes-requested |

退出 `changes-requested`:作者修改后重新提交评审,状态回到 `draft` → `reviewed`。

## 8. Definition-of-Done(工作项完成条件)

工作项完成当且仅当:

1. 主文档 status ≥ `reviewed`
2. Self-check 全部勾选
3. [`INDEX.md`](INDEX.md) 的"已完成设计文档清单"已追加该条
4. 若 `contract_impact=yes`,INDEX 决策日志已登记
5. 受本工作项影响的下游(按关系图)已被识别和标注

## 9. 评审准则(供 `fmt-design-review` 参考)

评审人按以下顺序检查并出具 verdict:

1. **结构合规**:§2 frontmatter + §3 章节 + §6 self-check 是否齐全
2. **依赖一致性**:upstream 列表的文档是否真的存在且 status ≥ reviewed
3. **退出条件覆盖**:00-design-plan.md 中的退出条件是否每条都有应答
4. **范围合规**:有无越权设计、有无遗漏
5. **契约影响**:对 firmware 契约的影响是否如实标注
6. **内容质量**:逻辑自洽、无矛盾、术语一致
7. **可下游可用性**:下游工作项能否仅凭本文档启动

- 1–6 任一不达标 → `changes-requested`,逐条列出问题。
- 仅第 7 项不达标 → 评审人据严重性裁决:阻塞性(下游确实无法启动)→ `changes-requested`;非阻塞性 → `pass` + non-blocking note。
- 1–7 全部达标 → `pass`,可附非阻塞改进建议。

## 10. 变更与回溯

文档发布后(status ≥ reviewed)的修改:

1. 在文末"变更日志"小节追加:`<日期> <修改者> <说明>`
2. 沿 [`01-design-relationships.md`](01-design-relationships.md) 出边给受影响下游加 `Affected by upstream change` 标记
3. 若变更触及 firmware 契约,INDEX 决策日志登记

## 11. 不在本规则范围

- 实现阶段的 Simulink/脚本编写规范(待实现阶段定)
- 代码生成配置规范(由 I4 设计项产出)
- 测试用例编写规范(由 H 区设计项产出)
