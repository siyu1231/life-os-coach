# ISA：Ideal State Artifact——把「做完」写成可验证的声明

本文件定义 Life Coach Agent 的 ISA（Ideal State Artifact）格式和使用方式。ISA 是一个轻量级的「完成标准」文档——它把模糊的「做好了」拆解为可验证的声明（Claims），让 Agent 和执行者都能清楚地知道：**怎样才算做完，怎么证明做完了**。

ISA 派生自 LifeOS 的核心原则：「没有证据 = 没做完」、「每个声明附带可证伪条件」、「声明只凭工具证据关闭」。但针对 Coach 场景做了简化——不需要代码级的严格性，保留核心精神即可。

## 目录

1. 什么是 ISA
2. ISA 格式模板
3. 何时需要 ISA
4. 何时不需要 ISA
5. ISA 在 Coach 场景中的使用方式
6. ISA 在 algorithm 循环中的角色
7. 示例 ISA 文档
8. 与滴答清单的关系
9. 与记忆系统的配合

---

## 一、什么是 ISA

### 1.1 定义

**Ideal State Artifact（ISA）**是一份结构化的文档，用于定义一项任务或目标的**理想完成状态**。它的核心是**可验证声明（Claims）**——每一条 Claim 都明确说明：

1. **做什么**：要达到的状态是什么。
2. **怎么验证**：用什么方式证明这个状态已经达到。
3. **验证来源**：证据来自哪里（工具返回值、文件内容、API 状态、用户确认等）。

### 1.2 为什么用 ISA

在日常任务管理中，「做完了」是一个极其模糊的概念：

- 「把设计稿改好」→ 什么叫「改好」？谁来判断？
- 「整理一下这周的复盘」→ 整理到什么程度算完成？
- 「推进项目进度」→ 推进了多少？怎么衡量？

ISA 的目标是**把完成标准从执行者的脑海里拽出来，写成任何人都能在事后验证的声明**。这样：

- Agent 知道自己要做到什么程度才算完成任务。
- 用户（或 CR 审查者）可以在事后对照 ISA Claims 逐条确认。
- 「没做完」变成可讨论的：不是「感觉没做完」，而是「Claim 3 的 verification_method 未通过」。

### 1.3 ISA 的四个核心要素

| 要素 | 含义 | 例子 |
|------|------|------|
| **Goal** | 用户的目标陈述（原文） | 「我想把本周的复盘整理出来」 |
| **Vision** | 背后的意图——完成这件事对用户为什么重要 | 「整理复盘是为了从这周的偏离中找到可复用的策略」 |
| **Claims** | 可验证的完成声明，每条含 verification_method | 「Claim 1: 本周 7 天的 daily memory 都已归档到 review-log.md」 |
| **Anti-claims** | 明确声明「什么样不算做完」或「不要做成什么样」 | 「不是写一篇长篇周记，不是追求文笔优美」 |

---

## 二、ISA 格式模板

### 2.1 完整模板

```markdown
# ISA: {任务名称}

> 创建日期：{YYYY-MM-DD}
> 状态：draft | active | done | superseded
> 关联：{滴答任务 ID（可选）}

## Goal

{用户的目标陈述原文，保持用户用语}

## Vision

{完成这件事的意图——它对用户为什么重要？解决了什么痛点或推进了什么方向？1-3 句话}

## Out of Scope

- {明确不在本次范围内的事项}
- {即使相关但本次不做的事情}
- {用户说「不要」的方向}

## Claims

每一条 Claim 包含：
- **声明**：达到什么状态才算这部分完成。
- **验证方式**：怎么证明（读文件、查 API、工具返回值、用户确认）。
- **验证来源**：证据出处的具体路径或 API 名称。

### Claim 1: {简短标题}

**声明**：{一句话描述完成状态}

**验证方式**：{可操作的具体验证步骤}

**验证来源**：{文件路径 / API 名称 / 用户确认}

### Claim 2: {简短标题}

...（一般 3-8 条 Claims）

## Anti-claims

{明确说明：怎样不算是做完了、不要做成什么样、哪些动作不算完成}

- {Anti-claim 1}
- {Anti-claim 2}
```

### 2.2 极简模板

对于简单任务，可以使用极简版本：

```markdown
# ISA: {任务名称}

**Goal**: {用户原话}

**Claims**:
- [ ] {Claim 1} → 验证：{verification_method}
- [ ] {Claim 2} → 验证：{verification_method}
- [ ] {Claim 3} → 验证：{verification_method}

**Anti-claims**: {1-2 句说明什么不算完成}
```

极简版省略 Goal/Vision/Out of Scope 的独立区块，Claims 用 checkbox 列表表达。适合 15 分钟内能完成的小任务。

### 2.3 Claims 的验证方式（verification_method）类型

| 验证类型 | 说明 | 示例 |
|---------|------|------|
| `file_read` | 读取指定文件，检查内容是否存在 | 「读取 `life-coach-data/reviews/review-log.md`，确认有本周的复盘条目」 |
| `api_query` | 调用外部 API 查询状态 | 「调用滴答查询 API，确认 task_id=xxx 的 `completed_time` 不为空」 |
| `grep_match` | 对文件执行文本匹配 | 「`grep "2026-07-29" life-coach-data/memory/daily/2026-07-29.md` 有输出」 |
| `user_confirm` | 需要用户确认（通常用于主观判断） | 「用户确认复盘中对『下午效率低』的归因准确」 |
| `tool_return` | 检查工具返回值 | 「邮件 API 返回 `errcode=0` 且 `message_id` 非空」 |
| `count_check` | 数量统计验证 | 「`life-coach-data/memory/daily/` 下有 7 个文件（本周每天一个）」 |
| `diff_check` | 对比修改前后的差异 | 「`git diff life-coach-data/planning/weekly-plan.md` 显示本周计划已更新」 |

---

## 三、何时需要 ISA

### 3.1 需要 ISA 的场景

满足以下**任意一条**，就应该创建 ISA：

1. **多步骤任务**：任务需要 2 个以上独立步骤才能完成。
2. **涉及外部工具**：任务包含文件写入、API 调用、数据库操作的任意一种。
3. **有模糊判断空间**：任务的完成标准不明确，「做好」需要定义（如「整理」「优化」「推进」）。
4. **有「不做」的边界**：用户明确说了某些方向不需要（Anti-claims 存在价值）。
5. **需要事后验证**：任务完成后可能需要回顾或审查（如周复盘、项目收尾）。
6. **多人或跨会话协作**：任务可能跨多次对话完成，需要传递完成标准。
7. **反复失败的任务**：同一任务之前被标记完成但实际未达到预期效果——说明完成标准需要重新定义。

### 3.2 判断口诀

> **「任何大于『看一眼就能回答『做完了吗』的任务，都值得一个 ISA。」**

看一眼就能回答的场景：

- 「今天天气怎么样？」→ 不需要 ISA。
- 「帮我把这条记录写到 daily memory」→ 不需要 ISA（单步、明确）。
- 「整理本周复盘」→ 需要 ISA（什么叫整理好了？）。

---

## 四、何时不需要 ISA

以下场景**不需要**创建 ISA：

| 场景 | 理由 | 例子 |
|------|------|------|
| **纯状态查询** | 没有「完成」的概念，只有信息获取 | 「今天有什么待办？」「本周哪天有会议？」 |
| **单步简单操作** | 动作和完成标准完全一致，无需额外定义 | 「记录一下：下午推进了设计稿」 |
| **纯情绪承接** | Coach 场景中只做情绪映照，不涉及任务推进 | 用户说「今天好累」，Agent 只做 empathy，不创建任务 |
| **简单问答** | 一问一答，无持续状态 | 「明天几点？」「滴答清单怎么用？」 |
| **信息转发** | 纯传递，无加工 | 「帮我把这段话发到邮件」 |
| **Cron 触达消息** | 每轮 Cron 消息是例行操作，完成标准由 Cron 配置定义 | 早安问候、状态检查、晚间复盘 |

### 4.1 边界判断

如果不确定是否需要 ISA，问自己：

> **「如果别人来做这个任务，ta 需要 ISA 才能判断自己做完没有吗？」**

如果答案是「是」，就创建 ISA。如果答案是「不需要，任务本身已经足够明确」，就不需要。

---

## 五、ISA 在 Coach 场景中的使用方式

### 5.1 核心精神保留，形式可简化

Coach 场景中的 ISA 不需要代码项目中的严格性。核心保留的是：

- **可验证**：Claims 的验证方式要说清楚，但不能变成形式主义。`user_confirm` 在 Coach 场景中是合法的验证方式（用户确认即可关闭 Claim）。
- **可讨论**：ISA 是 Agent 和用户之间关于「什么叫做完」的共识文件，不是合同条款。用户可以随时修改 Claims。
- **够用就好**：不要为每件小事写 ISA。ISA 的粒度以「帮助用户看清完成标准」为准，不以「覆盖所有可能性」为准。

### 5.2 Coach 场景下的 ISA 简化规则

| 正常 ISA 要求 | Coach 场景简化 |
|--------------|---------------|
| 所有 Claims 必须有客观的 verification_method | `user_confirm` 是可接受的验证方式 |
| Claims 必须穷举所有完成条件 | 只列最重要的 3-5 条 Claims，其余可被隐含 |
| Vision 必须精确描述业务意图 | Vision 可以用用户的自然语言写 1-2 句话 |
| Out of Scope 必须穷举 | 只列用户明确说「不要」的方向 |
| ISA 创建需评审 | Agent 可以主动草拟 ISA，用户确认即可 |

### 5.3 何时用 ISA、何时用 Coach Skill

| 场景 | 用什么 |
|------|-------|
| 用户说了一个模糊目标，需要澄清完成标准 | 创建 ISA |
| 用户需要被陪伴、疏导情绪，没有具体任务 | 用 Coach Skill（coaching-process），不创建 ISA |
| 用户在 Coaching 过程中产出了一个行动计划 | 为行动计划创建 ISA |
| 用户复盘后需要整理洞察 | 如果「整理」的边界模糊，创建 ISA |
| Cron 触达后用户回复了状态更新 | 通常不需要 ISA（状态更新是单步操作） |

### 5.4 ISA 的创建时机

在 Coach 对话中，以下信号提示该创建 ISA：

- 用户说了「我想……」但没有说怎么算做完。
- Agent 在 DECIDE 阶段标记了 `ISA_NEEDED`（见 `system/algorithm.md` 的 Step 3）。
- 一个任务跨越了多次会话（本次未完成，下次继续）。
- 用户在复盘时说「上次我以为做完了但其实没做完」——这说明完成标准需要重新定义。

---

## 六、ISA 在 algorithm 循环中的角色

ISA 是 algorithm 6 步循环中 GAP 和 VERIFY 阶段的标准来源：

```
algorithm 步骤              ISA 的角色
─────────────────────       ─────────────────────────────
CHECK（了解现状）           不直接使用 ISA
GAP（发现差距）             对照 ISA Claims 逐项判断 Current State
DECIDE（决定动作）           动作设计至少推进一条 Claim
ACT（执行动作）             按 Claim 的 verification_method 保留证据
VERIFY（工具验证）           verification_method 直接定义验证方式
LEARN（记录洞察）            未关闭的 Claim 是值得记录的差距模式
```

### 6.1 GAP 阶段的 ISA 使用

GAP 分析时，将 Current State 与 ISA 的 Claims 逐项对比：

| Claim 状态 | GAP 类型 | 动作方向 |
|-----------|---------|---------|
| Claim 全部 VERIFIED | `on_track` → 任务已完成 | 标记 ISA 状态为 `done` |
| Claim 部分 VERIFIED | `in_progress` | 决定是否继续推进剩余 Claim |
| Claim 全部 UNVERIFIED | `not_started` | 决定是否启动或推迟 |
| Claim 验证失败 | `blocked` | 记录阻塞原因 |

### 6.2 VERIFY 阶段的 ISA 使用

VERIFY 阶段直接使用 ISA Claims 中定义的 `verification_method`：

1. 读取 ISA 文档中的 Claims 列表。
2. 按每条 Claim 的 `verification_method` 执行验证。
3. 如果 `verification_method` 是 `user_confirm`，生成确认问题（如「你确认本周的复盘已经覆盖了全部 5 个工作日吗？」）。
4. 验证通过 → Claim 关闭（标记 `[x]`）。
5. 验证未通过 → Claim 保持 `[ ]`，记录未通过原因。

---

## 七、示例 ISA 文档

### 示例 1：轻量任务——整理本周复盘

```markdown
# ISA: 整理本周复盘（W30）

> 创建日期：2026-07-26
> 状态：active

**Goal**: 「帮我把这周的复盘整理一下，我想看看这周到底哪些事在重复卡住。」

**Vision**: 用户注意到最近几周反复在某些事情上偏离计划，想通过整理本周复盘找到其中的规律，而不是只记流水账。

**Claims**:
- [ ] Claim 1: 本周 5 个工作日的 daily memory 关键记录已合并到 `review-log.md`
  → 验证：`grep "2026-07-2[1-5]" life-coach-data/reviews/review-log.md` 有 5 条匹配
- [ ] Claim 2: 本周至少识别出 2 个「重复出现的偏离模式」
  → 验证：`review-log.md` 本周条目下存在 ## 重复模式 章节，且至少有 2 个子条目
- [ ] Claim 3: 每个偏离模式附带了「可能原因」和「下次实验方向」
  → 验证：user_confirm（用户确认每条模式都有原因和方向）
- [ ] Claim 4: 复盘结论中最重要的一条已写入 `memory/long-term.md`
  → 验证：`grep "W30复盘" life-coach-data/memory/long-term.md` 有输出

**Anti-claims**:
- 不是写一篇完整的长文章——重点是提取模式，不是文字优美。
- 不需要对每一天做详细描述——只需要关键偏离和洞察。
- 不需要给每一个问题都找到解决方案——未解决的问题标注「待观察」即可。
```

### 示例 2：轻度 Coach 任务——处理用户的拖延模式

```markdown
# ISA: 分析「下午开始困难」模式

> 创建日期：2026-07-24
> 状态：active

**Goal**: 「我最近每天下午都很难开始做事，帮我看看是怎么回事。」

**Vision**: 用户连续 5 天在下午 2 点后出现启动困难，想理清是精力问题、任务类型问题还是心理阻力问题，然后设计一个低成本的实验来验证。

**Claims**:
- [ ] Claim 1: 收集最近 7 天下午时段的实际行为数据（做了什么 vs 计划做什么）
  → 验证：`memory/daily/2026-07-{18..24}.md` 中均有下午时段的记录摘要
- [ ] Claim 2: 归类出至少 1 个「最可能的阻力因素」（精力低谷 / 任务恐惧 / 环境干扰 / 优先级模糊）
  → 验证：user_confirm（用户确认归因「有道理，贴近实际感受」）
- [ ] Claim 3: 设计 1 个低成本验证实验（一个下午即可完成的小改变）
  → 验证：实验设计包含「做什么、何时、多小、怎么判断有效」
- [ ] Claim 4: 实验记录写入 `memory/daily/{今天}.md`，标注为「实验:下午启动」
  → 验证：file_read 确认记录存在且标注正确

**Anti-claims**:
- 不是给出一个「下午效率提升的通用方案」——只做针对性分析。
- 不是挖童年创伤或做深度心理分析——聚焦最近 7 天的可观察行为。
- 不是要求用户承诺「以后下午都不拖延」——只设计一个实验。
```

### 示例 3：极简版——单步任务

```markdown
# ISA: 更新 daily-plan.md 的下午优先级

**Goal**: 「下午先把 B 项目的接口文档写完，A 项目往后推。」

**Claims**:
- [ ] `planning/daily-plan.md` 中下午时段的优先级已更新为 B 项目在前
  → 验证：file_read 确认 `planning/daily-plan.md` 的 ## 下午 区块第一条是 B 项目
- [ ] 用户已知晓修改
  → 验证：邮件/IM 确认消息已发送且 errcode=0

**Anti-claims**: 不是删除 A 项目——只是调整优先级，A 项目仍保留在今日计划中。
```

---

## 八、与滴答清单的关系

### 8.1 滴答任务映射为 ISA Claims

当一条滴答清单任务满足「需要 ISA」的条件时，该任务的完成标准可以由 ISA Claims 定义：

| 滴答任务字段 | 映射到 ISA |
|-------------|-----------|
| 任务标题 | ISA 的 Goal 或任务名称 |
| 任务描述 | ISA 的 Vision（如果用户写了意图） |
| 子任务 | ISA 的 Claims（每条子任务是一条 Claim） |
| 截止日期 | ISA 的创建日期上下文 |
| 标签/优先级 | 不影响 ISA 结构，但可影响 DECIDE 中的优先级排序 |

### 8.2 滴答任务不需要 ISA 的情况

大多数滴答任务不需要 ISA。ISA 只用于：

- 任务的完成标准无法直接从标题判断（如「整理」「优化」「推进」类动词）。
- 任务需要在 Agent 和用户之间传递完成标准。
- 任务跨多次会话或需要事后验证。

简单的滴答任务（如「买牛奶」「回复张三邮件」「9 点开会」）不需要 ISA——标题本身就是完成标准。

### 8.3 ISA 作为滴答子任务的扩展

如果滴答任务已有子任务，ISA Claims 可以直接引用子任务而不重复定义：

```markdown
**Claims**:
- [ ] Claim 1: 滴答子任务 1-3 全部标记完成
  → 验证：api_query 滴答任务详情，所有子任务 `completed_time` 不为空
- [ ] Claim 2: 生成的项目总结已写入 `projects/projects.md`
  → 验证：file_read
```

### 8.4 工具集成约定

ISA 涉及滴答清单的验证方式使用 `api_query` 类型：

```markdown
→ 验证：api_query（滴答 MCP `get_task` 确认 task_id={id} 的 status=completed）
```

具体 API 调用方式见 `integrations/dida-mcp.md`（待编写）。

---

## 九、与记忆系统的配合

ISA 文档本身存放在哪里、如何管理，遵循 `system/memory-system.md`（待编写）的规则：

### 9.1 存放位置

推荐将 ISA 文档存放在以下位置（与项目文件同在）：

```text
life-coach-data/
├── isa/
│   ├── active/        # 当前活跃的 ISA
│   │   ├── 2026-07-26-weekly-review-w30.md
│   │   └── 2026-07-24-afternoon-start-analysis.md
│   ├── done/          # 已完成的 ISA（归档）
│   │   └── 2026-07-20-weekly-review-w29.md
│   └── superseded/    # 被取代的 ISA（需求变更）
│       └── 2026-07-15-old-plan.md
```

### 9.2 ISA 生命周期

```
draft → active → done
              → superseded
```

- **draft**：Agent 草拟，尚未与用户确认。Claims 为初步假设。
- **active**：用户已确认（或对简单任务 Agent 可直接激活），Claims 生效。
- **done**：所有 Claims 的 verification_method 返回通过，ISA 归档。
- **superseded**：需求变更，原 ISA 被新版本取代。

### 9.3 写入前确认

涉及以下情况的 ISA 写入前需用户确认：

- ISA 包含对用户长期行为模式的判断（如「下午启动困难」的归因）。
- ISA 的 Claims 涉及修改用户长期画像或人生罗盘。
- ISA 包含需要用户本人才能验证的 Claim（`user_confirm` 类型）。

以下情况 Agent 可以直接写入：

- ISA 仅基于用户刚刚明确表达的目标和约束。
- ISA 的 Claims 全部为客观可验证（`file_read`、`api_query`、`tool_return` 等）。
- 用户说「帮我整理一下」，暗含授权 Agent 定义完成标准。

### 9.4 ISA 不替代 memory

ISA 是临时的完成标准文档，不是长期记忆。ISA 完成后：

- 完成状态归档到 `isa/done/`。
- 从中提取的洞察写入 `memory/long-term.md`（遵循 memory-system 的写入规则）。
- ISA 不保留用户状态事实——那是 `memory/daily/` 和 `profile/` 的职责。

---

## 文件完整性验证

### ISA 格式要素覆盖检查

```bash
grep -c "Goal\|Vision\|Out of Scope\|Claims\|Anti-claims" system/isa-system.md
```

预期：至少匹配 4（五个核心要素至少出现 4 个）。

### 验证方式类型覆盖检查

```bash
grep -c "verification_method\|验证方式\|验证类型" system/isa-system.md
```

预期：至少匹配 3。

### 与滴答清单关联检查

```bash
grep -c "滴答\|dida" system/isa-system.md
```

预期：至少匹配 2。

## References

- `system/algorithm.md`：执行循环，ISA 在 GAP 和 VERIFY 阶段的标准来源。
- `system/memory-system.md`（待编写）：记忆读写规则，ISA 的存放、生命周期和归档规则。
- `integrations/dida-mcp.md`（待编写）：滴答清单 MCP 集成，ISA Claims 映射到滴答子任务和 `api_query` 验证方式的实现。
- `engine/session-flow.md`：会话状态机，ISA 在 Phase 5（行动）和 Phase 6（闭环）中的使用。
- `_reference/life-coach/references/memory-system.md`：本地记忆系统参考，ISA 在长期记忆中定位的背景知识。
