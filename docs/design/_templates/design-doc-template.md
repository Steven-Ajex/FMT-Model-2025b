---
work_item: <ID>
title: <短标题>
upstream: [<ID>, <ID>]
contract_impact: <yes|no>
status: draft
authored_at: <YYYY-MM-DD>
last_reviewed_at:
reviewer_verdict: none
---

# <ID> <完整标题>

## 1. 目的

<一句话说明本文件解决什么设计问题。>

## 2. 范围

**在范围:**
- <项 1>
- <项 2>

**不在范围(由其他工作项处理):**
- <项 1> — 由 <ID> 处理
- <项 2> — 由 <ID> 处理

## 3. 依赖

| 上游 | 引用位置 | 用途 |
|---|---|---|
| <架构 v1 §N> | [链接](../../architecture/2026-05-05-fmt-model-architecture-v1.md) | <用途> |
| <ID> | [链接](../<area>/<id>-<slug>.md) | <用途> |

## 4. 设计内容

<本工作项的核心设计。子结构由 00-design-plan.md 中本工作项的"产出"和"退出条件"决定,作者自由组织。建议每节有清晰小标题。>

### 4.1 ...

### 4.2 ...

## 5. 已知风险与悬而未决问题

- **<风险/问题简述>**
  - 影响:<对哪些下游>
  - 处置:<推迟到哪个工作项 / 标 open / 已规避>

## 6. 退出条件复核

对照 [`00-design-plan.md`](../00-design-plan.md) 中本工作项"退出条件"逐条:

| # | 退出条件原文 | 本文档依据 | 状态 |
|---|---|---|---|
| 1 | <原文> | §<n> | 满足 |
| 2 | <原文> | §<n> | 满足 |

## 7. 下游影响

按 [`01-design-relationships.md`](../01-design-relationships.md) 本工作项的出边:

| 下游 | 关系类型 | 本文档为下游提供的输入 |
|---|---|---|
| <ID> | → / ⇢ / ↔ | <提供什么> |

## 8. 变更日志

| 日期 | 修改者 | 说明 |
|---|---|---|
| <YYYY-MM-DD> | <author> | 初稿 |

## Self-check

- [ ] frontmatter 完整,字段值合法
- [ ] 上游文档全部存在且 status ≥ reviewed,或为同 co-seal batch 兄弟(已在 §3 注明)
- [ ] 退出条件逐条复核完成,每条均给出依据
- [ ] 引用路径全部可点击访问
- [ ] 不存在 RULES §5 禁则中的内容
- [ ] 触及 firmware 契约者(contract_impact=yes)已在 INDEX 决策日志登记
- [ ] 镜像自 firmware 的契约已记录 firmware commit hash + 文件相对路径
- [ ] 下游影响已沿关系图识别完毕
- [ ] 文档不超出本工作项范围(无越权设计)
