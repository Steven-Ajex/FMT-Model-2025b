# FMT-Model-2025b 设计阶段执行 Playbook

最后更新:2026-05-05
配套文件:[`00-design-plan.md`](00-design-plan.md)、[`01-design-relationships.md`](01-design-relationships.md)、[`RULES.md`](RULES.md)

## 1. 目的

本文件定义"如何在多次 Claude 会话中、自主地、可复现地推进 33 个设计工作项",并保证:

- **单次对话独立性**:每个设计工作由一个独立子任务/会话完成,不依赖父对话上下文
- **平行评审**:每个完成的设计文档由独立 reviewer 子任务评审
- **决策可追溯**:所有完成项与决策都汇入 [`INDEX.md`](INDEX.md)

## 2. 角色

| 角色 | 实现方式 | 职责 |
|---|---|---|
| **Orchestrator** | 主对话(用户与之交互) | 选择下一波工作项、派发子任务、收集结果、更新 INDEX |
| **Author** | 子代理 + `fmt-design-author` skill | 撰写一个设计工作项的主文档 |
| **Reviewer** | 子代理 + `fmt-design-review` skill | 审阅一个设计文档,出具 verdict |
| **Plan Auditor** | 后台子代理 | 一次性整体审查开发计划/架构;独立于具体工作项 |

## 3. 派发模式

### 3.1 Author 派发(单工作项)

```text
触发:Orchestrator 决定启动某工作项 X
工具:Agent(subagent_type=general-purpose)
模式:foreground 或 background(取决于是否需要等待)
```

**Brief 模板**(自包含,Author 子代理无需父对话上下文):

```text
You are authoring a design document for FMT-Model-2025b work item <ID>.

Context (read these in order):
1. RULES: /Users/stevenajex/FMT/FMT-Model-2025b/docs/design/RULES.md
2. Design plan (find your work item row): /Users/stevenajex/FMT/FMT-Model-2025b/docs/design/00-design-plan.md
3. Relationships (find your upstream/downstream): /Users/stevenajex/FMT/FMT-Model-2025b/docs/design/01-design-relationships.md
4. Architecture v1: /Users/stevenajex/FMT/FMT-Model-2025b/docs/architecture/2026-05-05-fmt-model-architecture-v1.md
5. Skill: /Users/stevenajex/FMT/.claude/skills/fmt-design-author/SKILL.md
6. Template: /Users/stevenajex/FMT/FMT-Model-2025b/docs/design/_templates/design-doc-template.md

Upstream snapshot (authoritative — DO NOT discover via INDEX, which may race):
<orchestrator inserts an explicit list here, e.g.>
  - 架构 v1 — /Users/stevenajex/FMT/FMT-Model-2025b/docs/architecture/2026-05-05-fmt-model-architecture-v1.md (always-on seed input)
  - <ID> — <abs path> (status: reviewed at <date>)
  - <ID> — <abs path> (status: reviewed at <date>)
Co-seal batch (if applicable):
  - <batch name>, siblings: [<ID>, <ID>, <ID>]
  - You MAY reference siblings even if they are still in draft, per RULES §6 co-seal allowance.

Task:
- Author work item <ID>: <name from 00-design-plan>
- Follow RULES.md exactly (frontmatter, sections, Self-check, naming).
- Write the file to: /Users/stevenajex/FMT/FMT-Model-2025b/docs/design/<area>/<id>-<slug>.md
- Set status: draft
- DO NOT read INDEX.md to discover upstreams; trust the snapshot above.
- DO NOT update INDEX.md — that's the orchestrator's job.
- DO NOT start any other work item.
- Return a structured summary: file path, status, any blockers, any contract impact.

Constraints:
- No code, no .slx, no .m executable scripts (pseudocode only, marked).
- Do not redefine bus/enum/params if they live in B-contracts (reference them).
- If an upstream listed in the snapshot is missing on disk, STOP and report — do not invent.
- If you would need an upstream NOT in the snapshot, STOP and report; the orchestrator will resolve dispatch order.
```

### 3.2 Reviewer 派发(单文档)

```text
触发:Author 完成 → Orchestrator 立即派发 Reviewer
工具:Agent(subagent_type=general-purpose)
模式:foreground(需 verdict 决定下一步)
```

**Brief 模板**:

```text
You are reviewing a draft design document for FMT-Model-2025b.

Context:
1. RULES (review criteria in §9): /Users/stevenajex/FMT/FMT-Model-2025b/docs/design/RULES.md
2. Design plan (find this work item's exit criteria): /Users/stevenajex/FMT/FMT-Model-2025b/docs/design/00-design-plan.md
3. Skill: /Users/stevenajex/FMT/.claude/skills/fmt-design-review/SKILL.md
4. Architecture v1: /Users/stevenajex/FMT/FMT-Model-2025b/docs/architecture/2026-05-05-fmt-model-architecture-v1.md
5. INDEX: /Users/stevenajex/FMT/FMT-Model-2025b/docs/design/INDEX.md

Document under review:
- Path: <full path>
- Work item ID: <ID>

Task:
- Apply RULES §9 review criteria (1–7).
- Output a structured review report:
  * Verdict: pass | changes-requested
  * For each criterion 1–7: status + evidence
  * If changes-requested: numbered list of issues with file:line citations
  * Optional non-blocking suggestions
- DO NOT edit the document.
- DO NOT update INDEX.
- Return the report only.
```

### 3.3 Plan Auditor 派发(整体审查,平行)

```text
触发:Orchestrator 在设计阶段早期启动一次,运行于 background
工具:Agent(subagent_type=general-purpose, run_in_background=true)
模式:background(独立于具体工作项,完成后回报)
```

**Brief 模板**:

```text
You are an independent auditor for the FMT-Model-2025b development plan.

Read these in full:
1. /Users/stevenajex/FMT/FMT-Model-2025b/docs/architecture/2026-05-05-fmt-model-architecture-v1.md
2. /Users/stevenajex/FMT/FMT-Model-2025b/docs/design/00-design-plan.md
3. /Users/stevenajex/FMT/FMT-Model-2025b/docs/design/01-design-relationships.md
4. /Users/stevenajex/FMT/FMT-Model-2025b/docs/design/RULES.md
5. /Users/stevenajex/FMT/FMT-Model-2025b/docs/design/INDEX.md

Find:
- Gaps: design areas the plan does not cover but should
- Contradictions: between architecture v1 and design plan
- Sequencing issues: dependency edges in 01-design-relationships that conflict with §6 wave order
- Missing exit criteria: work items whose criteria are too vague to verify
- Risk coverage: risks in architecture §17 not addressed by any work item
- Consolidation opportunities: items that should merge
- Split opportunities: items too coarse to deliver in one doc
- Contract risk: places where firmware contract impact may be silently missed

Output:
- Write the report to: `/Users/stevenajex/FMT/FMT-Model-2025b/docs/design/_audit/<YYYY-MM-DD>-plan-audit.md` (create `_audit/` if missing)
- Word budget: target ≤ 2500 words
- Format (markdown):
  - Executive summary (≤ 8 bullets)
  - Findings table: id | category | severity (critical/major/minor) | file:section | description | recommended action
  - Recommended Plan Diffs: per-file textual descriptions (not actual diff blocks) for 00-design-plan / 01-design-relationships / 02-execution-playbook / RULES / architecture v1 (if needed)
  - Open Questions for the User
  - Methodology Notes (what was verified vs inferred; files unread)

Do NOT modify any file other than your audit report. Return a brief summary (≤ 200 words) of top findings + the audit report path.
```

### 3.4 Co-seal batch dispatch

For batches identified in §5(协同设计对/组),the orchestrator picks one of two patterns:

**Pattern A — Single-brief authoring**(适合紧密耦合,如 (D6, E1b)): one Author subagent authors all sibling docs together; reviewer evaluates them as a set.

**Pattern B — Parallel authoring with shared review**(适合多人/多代理工作流,如 (B1, B2, B3)): siblings dispatched in parallel; orchestrator collects all drafts; one Reviewer reviews the batch as a unit, looking for cross-document consistency. RULES §6 self-check explicitly allows referencing same-batch siblings still in draft.

In either pattern, no sibling reaches `reviewed` status until **all** siblings in the batch pass review.

## 4. 推荐执行节拍

按 [`01-design-relationships.md §6`](01-design-relationships.md) 的 12 个 Wave 顺序推进。每个 Wave 内的工作项可并行 Author。

```text
for each Wave w in [Wave 1 .. Wave 12]:
    in parallel for each work item X in w:
        dispatch Author(X)         (foreground 或 background;同 Wave 内允许并行)
    await all Authors of w
    in parallel for each completed draft:
        dispatch Reviewer(draft)
    await all Reviewers
    for each draft:
        if verdict = pass:
            update INDEX completed list (Orchestrator 操作)
            if contract_impact = yes:
                update INDEX decision log
        else:
            re-dispatch Author with reviewer report attached
    proceed to Wave w+1
```

## 5. Orchestrator 在主对话中的责任

每次会话开始时:

1. 读 [`INDEX.md`](INDEX.md) 已完成清单 → 确定当前进度
2. 读 [`01-design-relationships.md §6`](01-design-relationships.md) → 确定下一个待启动 Wave
3. 检查依赖:Wave 内工作项的所有上游是否 status ≥ reviewed
4. 派发(§3 模式)
5. 等子代理回报后:
   - 更新 INDEX 已完成清单
   - 若 contract_impact=yes,更新 INDEX 决策日志
   - 若 reviewer 打回:把报告作为 brief 附件重派 Author

主对话**不**自己写设计文档主体 — 那是 Author 的职责。主对话的工作是:**调度 + 收集结果 + 更新索引**。

## 6. 单次对话独立性保证

- 每个 Author / Reviewer brief 用绝对路径列出所有需要读的文件
- Brief 内不假设子代理见过父对话历史
- Brief 末尾要求结构化返回,父对话仅消费返回内容
- 子代理产出的设计文档本身遵守 RULES.md 自包含(§3 依赖章节列上游),无需访问父对话即可被未来其他子代理消费

## 7. 失败模式与处置

| 失败 | 处置 |
|---|---|
| Author 因上游缺失停止 | Orchestrator 检查 Wave 顺序;若是真依赖缺口,先补上游;若是 Author 误判,在 brief 中澄清后重派 |
| Reviewer 反复 changes-requested | 第三次仍未通过时,Orchestrator 把所有评审报告 + 草稿汇总,提交用户人工裁决 |
| Plan Auditor 报告与计划严重冲突 | 暂停 Wave 派发,先由用户对计划做修订,然后从 INDEX 当前进度恢复 |
| 子代理超时/中断 | Orchestrator 重试一次;仍失败则记入 INDEX 风险栏并跳过到下一项 |

## 8. 与现有 superpowers skills 的关系

| 现有 skill | 在本 playbook 中的用法 |
|---|---|
| `superpowers:writing-plans` | 仅用于实现阶段;设计阶段不直接使用 |
| `superpowers:requesting-code-review` | 替代为本 playbook 的 Reviewer 模式(评审对象是设计文档,不是代码) |
| `superpowers:dispatching-parallel-agents` | 适用于同 Wave 内多 Author 并行 |
| `superpowers:subagent-driven-development` | 适用于实现阶段;设计阶段类比但对象不同 |

## 9. 不在本 playbook 范围

- 实现阶段如何派发(待设计阶段结束后另立 playbook)
- Codegen / SIL / SIH 流程(由 I4、H 区设计项产出)
- 多机型扩展时的派发策略(单机型设计完成后再扩展)
