# 执行循环与推理框架

本文件定义 Life Coach Agent 的核心执行循环——一个派生自 LifeOS 理念的 6 步推理-行动闭环：**CHECK → GAP → DECIDE → ACT → VERIFY → LEARN**。

本文件是 `engine/session-flow.md` 中 6 阶段状态机（采集→触达→接收→理解→行动→闭环）在**推理层面**的对偶：session-flow 定义「消息何时收发、状态如何转移」；algorithm 定义「Agent 在每一步如何思考、读什么、判断什么、输出什么」。

## 目录

1. 核心哲学
2. 六步循环总览
3. 每步详细说明
4. 与 session-flow 的对偶关系
5. 与记忆系统的配合
6. 与 ISA 系统的配合
7. 核心原则
8. AI 工具实现指导

---

## 一、核心哲学

### 1.1 Current → Ideal 爬山

LifeOS 的底层世界观：**每个任务都是一次从 Current State（当前状态）向 Ideal State（理想状态）的爬山运动**。Agent 的职责不是替用户爬山，而是：

- 帮助用户看清 Current State 的真实面貌（而非用户以为的样子）。
- 明确 Ideal State 的可验证标准（而非模糊的「做好」）。
- 找到从 Current 到 Ideal 的下一步动作（而非给出整个山的路线图）。
- 用工具证据验证是否真的在靠近（而非靠感觉说「差不多了」）。

### 1.2 证据驱动

**「没有证据 = 没做完」**。声明一件事「完成了」的唯一依据是工具返回的确认信号：

- 文件写入 → 读回校验。
- 邮件发送 → API 返回 `errcode=0` + `message_id`。
- 滴答清单任务标记 → 查询 API 确认状态已变更。
- 计划更新 → 读回文件确认内容已写入。

在收到工具确认信号之前，任何声明都只是假设。

### 1.3 声明即附带可证伪条件

**每个声明都附带其可证伪条件**。说「任务已完成」的同时，必须说明「怎么验证它已完成」。例如：

- 「数据已写入 memory/daily/2026-07-29.md」→ 验证方式：读取该文件，确认记录存在。
- 「邮件已发送给用户」→ 验证方式：调用 `get_mail_content` API，确认 `send_time` 在 30 秒内。
- 「待办已标记完成」→ 验证方式：调用滴答查询 API，确认该 task 的 `completed_time` 不为空。

### 1.4 声明只凭工具证据关闭

**声明（Claim）只在工具证据确认后才关闭**。在证据到达之前：

- VERIFY 阶段未通过 → Claim 保持 `open`。
- 工具调用失败 → Claim 保持 `open`，记录失败原因。
- 工具返回不一致 → Claim 保持 `open`，标记 `需要人工核对`。

---

## 二、六步循环总览

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│   ┌─────────┐   ┌─────────┐   ┌─────────┐                      │
│   │ 1.CHECK │──▶│ 2.GAP   │──▶│ 3.DECIDE│                      │
│   │ 了解现状 │   │ 发现差距 │   │ 决定动作 │                      │
│   └─────────┘   └─────────┘   └─────────┘                      │
│        ▲                             │                           │
│        │                             ▼                           │
│        │                      ┌─────────┐                        │
│        │                      │ 4.ACT   │                        │
│        │                      │ 执行动作 │                        │
│        │                      └─────────┘                        │
│        │                           │                             │
│        │                           ▼                             │
│   ┌─────────┐               ┌─────────┐                         │
│   │ 6.LEARN │◀──────────────│ 5.VERIFY│                         │
│   │ 记录洞察 │               │ 工具验证 │                         │
│   └─────────┘               └─────────┘                         │
│        │                                                         │
│        └─────────────────────────────────────────────────────────│
│                         （下一轮循环）                             │
└──────────────────────────────────────────────────────────────────┘
```

### 步骤速查表

| 步骤 | 英文 | 核心问题 | 主要输入 | 主要输出 |
|------|------|---------|---------|---------|
| CHECK | Check Current State | 现在实际是什么状态？ | memory、calendar、Dida、上次 LEARN 的洞察 | 一份 Current State 摘要 |
| GAP | Gap Analysis | 实际 vs 期望的差距在哪？ | Current State 摘要 + 本期计划/期望 | GAP 报告（差距类型 + 严重程度 + 状态信号） |
| DECIDE | Decide Action | 基于差距，应该做什么？ | GAP 报告 + 用户画像 + 历史响应模式 | Action Plan（动作类型 + 参数 + 优先级） |
| ACT | Act | 执行选定的动作 | Action Plan | 执行记录（工具调用、参数、返回值） |
| VERIFY | Verify | 工具确认动作确实生效了吗？ | 执行记录 + 工具返回值 | Verification Report（通过/未通过/待确认） |
| LEARN | Learn & Record | 这次循环值得记住什么？ | Verification Report + 全程上下文 | 洞察写入 memory |

---

## 三、每步详细说明

### Step 1: CHECK——了解当前状态

**核心问题**：现在实际是什么状态？用户的 Current State 到底长什么样？

**读取什么**：

根据触发场景按需读取，不做全量加载：

| 触发场景 | 读取位置 |
|---------|---------|
| Cron 触达（工作模式） | `profile/user.md`、`memory/daily/{today}.md`、`memory/daily/{yesterday}.md`、`planning/commitments.md`、`planning/carryover.md`（未完成项台账）、`planning/daily-plan.md`、外部日历 API、外部待办 API |
| Cron 触达（休息日默认模式） | `profile/user.md`、`memory/daily/{yesterday}.md`、`planning/commitments.md`（筛选 rest_day 声明 + 今日到期/每日 step + 未来 7 天里程碑）、不读取 daily-plan.md（休息日不写强日计划） |
| Cron 触达（纯休息模式） | `profile/user.md`、`memory/daily/{yesterday}.md`、`planning/commitments.md`（仅筛选 rest_day 声明和日程事件类承诺）、不读取 daily-plan.md |
| 用户主动回复 | `memory/daily/{today}.md`、上一次会话的 LEARN 记录、`memory/long-term.md`（如需跨日上下文） |
| 复盘触发 | `planning/weekly-plan.md`、`planning/daily-plan.md`、`reviews/review-log.md`、`projects/projects.md`、近 7 日 `memory/daily/*.md` |
| E1 晚间节点（20:00） | `profile/user.md`、`planning/commitments.md`（四类标签 pending step/goal）、TELOS 长期目标晚间可执行步骤、滴答清单、习惯追踪、`planning/daily-plan.md`（17:30 收尾结论）、近 7 日 `memory/daily/*.md`（晚间四类覆盖统计） |
| E2 晚间节点（21:00） | 第一块执行结果（`memory/daily/{today}.md` 晚间段）、E1 填充方案、用户回复文本（用于类别判定） |
| E3 晚间节点（22:00） | 第一、二块执行结果（`memory/daily/{today}.md` 晚间段）、E2 填充/调整、用户回复文本（用于类别判定） |
| 深度问题 | `profile/life-compass.md`、`profile/user.md`、`memory/long-term.md`、相关 skill 的 cases 和 toolkit |

读取策略遵循 `system/memory-system.md` 的「启动时读取什么」规则。

**做什么**：

1. 收集本次决策所需的 Current State 数据。
2. 区分「事实」（工具返回的数据）和「解读」（Agent 对数据的理解）。
3. **先回顾上一个执行块**（Cron 触达场景适用）：读取上一个执行块的执行数据（`memory/daily/{today}.md` 中上一次记录的完成状态 + `planning/daily-plan.md` 的计划进度），确认上一个块实际发生了什么——而不是盲目基于旧计划做判断。如果上一个块没有数据（例如本日第一个 Cron 节点），读取昨日复盘数据作为间接上下文。
4. 将收集到的数据整理为一份 Current State 摘要，包含：
   - 时间上下文：当前时间点、日类型（workday/weekend/holiday）、**最终模式（workday/rest_day_default/pure_rest）**。
   - 上一块回顾：上一个执行块的计划 vs 实际、精力线索、偏离情况。
   - 用户状态：今日已完成事项、今日计划 vs 实际进度、精力线索、情绪信号。
   - 环境上下文：今日日程、待办列表、天气（仅早安节点）。
   - 历史线索：昨日偏离、上次 LEARN 中标记的「待观察」模式。
5. **拉取候选动作（承诺优先填充）**：从 `planning/commitments.md`（四类标签的 pending step/goal）、昨日复盘未完成项、滴答清单（今日任务）、TELOS 长期目标晚间可执行步骤、习惯缺口（本周未达标习惯）、邮件提取的未安排意图，按优先级拉取候选动作，自动填充计划草案并呈现用户确认（详见 `engine/cron-system.md` 的「承诺优先填充」机制）。
6. **晚间四类覆盖统计（仅 E1 节点）**：读取近 7 天 `memory/daily/*.md` 的「晚间」段，统计精进/休闲/锻炼/副业四类覆盖次数，标记连续缺失 2 天及以上的类别作为覆盖缺口（详见 `engine/cron-system.md` 的「晚间四类覆盖追踪」）。

**输出**：一份结构化的 Current State 摘要。

**与 session-flow 的对应**：session-flow 的 Phase 1（采集）是本步骤的消息通道版本。CHECK 的职责更广——不仅用于 Cron 触达前的上下文收集，也用于用户回复后的状态理解、复盘时的历史回溯。

---

### Step 2: GAP——发现差距

**核心问题**：用户的 Current State 和 Ideal State（或本期期望状态）之间有什么差距？

**读取什么**：

- Step 1 输出的 Current State 摘要。
- 本期计划/期望（来自 `planning/daily-plan.md`、`planning/weekly-plan.md`、`planning/commitments.md`、ISA 文档中的 Claims）。
- 用户历史模式（来自 `memory/long-term.md`、`reviews/review-log.md`）。

**做什么**：

1. **对比**：将 Current State 与本期计划/期望逐项对比。
2. **分类差距**：将发现的差距归入以下类型：

| 差距类型 | 特征 | 信号示例 |
|---------|------|---------|
| `on_track` | 实际接近计划，偏差在可接受范围 | 任务完成率 >= 80%，无异常延迟 |
| `minor_deviation` | 有小幅偏离，但趋势可控 | 某项任务延迟 < 2 小时，用户自行调整中 |
| `significant_deviation` | 明显偏离计划，需要关注 | 核心任务未启动，半天过去了无进展 |
| `emotional_blocker` | 非任务层面，用户表达了情绪阻力 | 「不想做」「好累」「卡住了」 |
| `plan_unrealistic` | 计划本身容量超出实际可用时间/精力 | 安排了 8 小时任务但只有 4 小时可用窗口 |
| `context_shift` | 外部环境变化导致原计划不再适用 | 突发会议、紧急事务、身体不适 |
| `no_plan` | 该时段有计划但用户未制定 | 周计划为空，日计划未填写 |
| `commitment_overdue` | 承诺或长期目标 step 已过期或即将到期 | commitments.md 中目标日期在今天或之前的 pending 项 |
| `carryover_stuck` | 未完成项连续顺延 ≥3 次且用户未表态，已降权但任务仍挂起 | carryover.md 中处置状态为「滞留」的条目；顺延 ≥2 次的「待处置」条目 |
| `rest_day_override` | 用户声明了工作日为纯休息（请年假等），原工作日计划需调整 | 当天有 rest_day 声明且当日 mode 为 pure_rest |
| `rest_day_step_available` | 休息日默认模式下有今日到期或「每日」step 可推进 | commitments.md 中目标日期 == today 或 == "每日" 的 step |
| `evening_category_gap` | 近 7 天晚间某类别连续缺失，覆盖失衡 | 晚间四类覆盖统计显示某类连续 2 天及以上未出现 |
| `task_commitment_detected` | 用户表达了新的任务意图/承诺，但尚未写入待办或安排时间 | 邮件回复匹配「我要/得/准备/打算/应该做 X」「下次/明天/周末 X」等模式 |

3. **提取状态信号**：从差距中提取影响后续决策的信号：

| 信号 | 含义 | 影响 |
|------|------|------|
| `energy_dip` | 用户在特定时段反复出现精力低谷 | 调整该时段的任务类型或容量 |
| `avoidance_pattern` | 某类任务被持续推迟 | 可能涉及 procrastination-execution skill |
| `overcommitment` | 计划容量持续超出实际完成量 | 需要在下次计划时降低容量预估 |
| `disengagement` | 用户连续未回复或回复极简 | 可能需要调整触达频率或风格 |

**输出**：GAP 报告，包含：

- 差距列表（每项标注类型、严重程度、关联信号）。
- 优先级排序：最需要立即回应的差距排在前面。
- 如果未发现明显差距（全部 `on_track`），输出 `GAP_CLEAR` 标记。

**与 session-flow 的对应**：session-flow 的 Phase 4（理解）中的分类结果（`status_update` / `emotional_signal` / `modification_request` / `deep_question`）是 GAP 分析在用户回复场景下的特化。例如 `emotional_signal` 对应 GAP 类型中的 `emotional_blocker`。

---

### Step 3: DECIDE——决定动作

**核心问题**：基于差距分析的结果，现在应该做什么？

**读取什么**：

- Step 2 输出的 GAP 报告。
- `profile/user.md`：用户沟通偏好、精力窗口、雷区（避免在用户低能量时发送挑战型消息）。
- 上一次同类型决策的结果（来自 `memory/long-term.md`）：上次类似差距用了什么动作，效果如何。

**做什么**：

1. **匹配动作类型**：根据 GAP 报告中的差距类型，选择动作类别：

| 差距类型 | 动作类别 | 说明 |
|---------|---------|------|
| `on_track` | `status_ack` | 正向反馈，记录状态，不做干预 |
| `minor_deviation` | `gentle_nudge` | 轻量提醒或询问，不施加压力 |
| `significant_deviation` | `coach_intervention` | 启动对应 coach skill（procrastination-execution / energy-management） |
| `emotional_blocker` | `emotional_support` | 优先情绪承接，降低期望，可能推迟任务推进 |
| `plan_unrealistic` | `plan_adjustment` | 帮助用户调整计划容量或优先级 |
| `context_shift` | `replan` | 根据新上下文重新安排 |
| `no_plan` | `planning_assist` | 启动 planning skill 帮助用户制定计划 |
| `commitment_overdue` | `commitment_followup` | 检查 commitments.md 中已过期或即将到期的承诺，提醒用户确认（延期/取消/完成） |
| `carryover_stuck` | `carryover_disposition` | 顺延 ≥2 次 → 17:40 聚焦处置邮件（继续延/取消/拆小/降级）；≥3 次未表态 → 标记滞留、09:30 降权进候补池、挂入周日 23:00 滞留清单批量处置；已滞留项不重复每日询问 |
| `rest_day_override` | `plan_adjustment` | 工作日被 rest_day 覆盖，将该日的原计划 task 和 commitment 关联起来——询问用户是延期还是取消 |
| `rest_day_step_available` | `gentle_nudge` | 休息日默认模式，轻量提醒有可推进的 step——不追进度，提供跳过选项 |

2. **优先级排序**：

   当多个差距同时存在时，按以下优先级处理：
   1. 安全相关（自伤/伤人/危机信号）→ 立即转介，不继续循环。
   2. 强情绪信号（用户明显痛苦）→ 优先情绪承接。
   3. 严重偏离（核心任务未启动、已影响今日目标）→ 启动 coach skill。
   4. 计划容量问题 → 调整计划。
   5. 轻微偏离 → 轻量提醒或仅记录。

3. **动作参数化**：为选定的动作填充具体参数：

   - 使用哪个 skill（或不需要 skill，Agent 直接处理）。
   - 回复用户还是仅记录（参考 session-flow Phase 5 的 Action 策略）。
   - 消息风格：`warm`（温暖）/ `brief`（简洁）/ `challenge`（挑战）。
   - 是否需要用户回复（开放式问题 vs 陈述式确认）。

4. **检查 ISA**：如果当前动作涉及「完成任务」的声明，检查是否已有 ISA 文档定义该任务的完成标准。如果没有 ISA 且任务复杂度需要，标记 `ISA_NEEDED`。

**输出**：Action Plan，包含：

- 选定的动作类别。
- 动作参数（skill、消息风格、是否回复、回复内容框架）。
- ISA 引用（如适用）。
- 决策依据（引用 GAP 报告中的具体差距）。

**与 session-flow 的对应**：session-flow 的 Phase 4 尾部（分类结果使用）和 Phase 5（行动）是本步骤的消息通道版本。DECIDE 是 session-flow 中「分类 → Action 路由」的推理通用化——不仅用于用户回复分类后的路由，也用于 Cron 主动触达前的决策。

---

### Step 4: ACT——执行动作

**核心问题**：执行选定的动作，产生实际效果。

**做什么**：

根据 DECIDE 的输出，执行以下一种或多种操作：

#### 4.1 状态同步（status_ack / gentle_nudge）

- 将用户状态写入 `memory/daily/{today}.md`。
- 如果涉及计划进度，更新 `planning/daily-plan.md` 的实际完成状态。
- 如果只是 Cron 触达，不追加额外操作（仅发送 Cron 消息模板本身）。

#### 4.2 Coach 介入（coach_intervention）

- 调用对应 coach skill 的方法论（参考 `coach/skills/` 下的 SKILL.md 和 references/）。
- 在邮件回复中给出 coach 引导（遵循「一封邮件一个问题」原则，见 `engine/email-protocol.md`）。
- 不试图在单封邮件中完成整个 coaching process——仅给用户一个入口。

#### 4.3 计划调整（plan_adjustment / replan / planning_assist）

- 更新 `planning/daily-plan.md` 或 `planning/weekly-plan.md`。
- 更新 `projects/projects.md` 中的优先级或状态。
- 如果需要用户确认（如大幅调整），先发送确认请求再写入。

#### 4.4 情绪支持（emotional_support）

- 使用 `coach/coaching-process.md` 的情绪承接方法。
- 遵循 session-flow Phase 5 Action B 的约束：先映照、正常化、降低期望、不催促。
- 不将情绪信号直接写入 `profile/user.md`（需用户确认）。

#### 4.5 特殊指令执行

- 识别用户回复中的特殊指令（见 `engine/email-protocol.md` 的「特殊指令集」）：暂停/恢复邮件、暂停一天、请求总结等。
- 执行对应操作：修改配置标记、生成异步任务等。
- 特殊指令直接执行，不进入 coach 推理。

**执行约束**：

- 所有外部工具调用（API、文件写入）必须记录调用参数和返回值。
- 工具调用失败时，降级处理而非阻塞整个循环（降级策略见 `system/memory-system.md` 的「工具失败可降级」原则）。
- 涉及修改用户长期画像、项目优先级、外部工具（滴答清单、日历）的写入，执行前需用户确认（见 `system/memory-system.md` 的「写入前确认」规则）。

**输出**：执行记录，包含每项操作的：

- 操作类型和参数。
- 工具调用结果（成功/失败、返回码、message_id 等）。
- 时间戳。

**与 session-flow 的对应**：session-flow 的 Phase 5（行动）是本步骤的消息通道版本。ACT 的执行范围更广——不仅包括回复用户，还包括本地文件写入、外部 API 调用、coach skill 调用。

---

### Step 5: VERIFY——工具验证

**核心问题**：工具确认刚才的动作确实生效了吗？

**核心原则**：**没有证据 = 没做完**。在收到工具确认信号之前，任何声明都只是假设。

**做什么**：

对 Step 4 中每项操作进行验证：

#### 5.1 文件写入验证

| 操作 | 验证方式 | 通过标准 |
|------|---------|---------|
| 写入 `memory/daily/{today}.md` | 读取该文件，检查对应行是否存在 | 目标内容出现在文件中 |
| 更新 `planning/daily-plan.md` | 读取该文件，检查状态字段或完成标记 | 更新的字段值与写入值一致 |
| 更新 `profile/user.md` | 读取该文件，检查对应章节 | 更新的内容正确写入 |
| 更新 `projects/projects.md` | 读取该文件，检查项目状态 | 状态变更已反映 |

#### 5.2 邮件发送验证

| 操作 | 验证方式 | 通过标准 |
|------|---------|---------|
| 发送邮件 | 检查 API 返回值 `errcode == 0` 且 `message_id` 非空 | errcode=0 |
| 获取用户回复 | 检查 `mailid` 不在去重集合中，内容提取成功 | `mailid` 唯一，纯文本内容非空 |
| 特殊指令确认回复 | 同上 | 确认消息发送成功 |

#### 5.3 外部工具验证

| 操作 | 验证方式 | 通过标准 |
|------|---------|---------|
| 滴答清单创建/更新任务 | 调用查询 API 确认任务状态 | 返回的任务字段与操作一致 |
| 日历事件创建 | 调用日历查询 API 确认事件存在 | 返回的事件时间/标题匹配 |

#### 5.4 验证结果

| 结果 | 含义 | 后续动作 |
|------|------|---------|
| `VERIFIED` | 所有验证通过 | 进入 LEARN |
| `PARTIALLY_VERIFIED` | 部分通过，部分失败 | 标记失败项，重试一次。二次失败则进入 LEARN（记录为 `action_failed`） |
| `UNVERIFIED` | 全部失败 | 进入 LEARN（记录为 `action_failed`），如果是关键操作则标记 `NEEDS_HUMAN_ATTENTION` |

**输出**：Verification Report，包含：

- 每项操作的验证状态（VERIFIED / PARTIALLY_VERIFIED / UNVERIFIED）。
- 失败项的详细错误信息。
- 是否重试及重试结果。

**与 session-flow 的对应**：session-flow 的 Phase 6（闭环）中的 Step 2（写入 Memory）和 Step 3（更新消息日志）依赖于本步骤的验证结果。如果 VERIFY 未通过，session-flow 闭环时记录的是未确认的操作而非已确认的结果。

---

### Step 6: LEARN——记录洞察

**核心问题**：这次循环中，有什么值得记住的？下次怎么做得更好？

**做什么**：

#### 6.1 提取洞察

从本次循环的全程上下文中提取可复用的洞察：

| 洞察类型 | 来源 | 写入位置 |
|---------|------|---------|
| 用户状态事实 | CHECK 中收集的今日事实 | `memory/daily/{today}.md` |
| 差距模式 | GAP 中发现的重复偏离 | `memory/long-term.md`（标注「待确认」） |
| 动作有效性 | ACT + VERIFY：哪种动作对哪种差距有效 | `memory/long-term.md`（标注「已验证策略」） |
| 精力/执行线索 | 多次出现的精力低谷或启动困难模式 | `memory/long-term.md`（标注「待确认模式」），若已稳定则候选写入 `profile/user.md` |
| 本次循环摘要 | 全程概要 | `memory/daily/{today}.md` 的触达记录部分 |

#### 6.2 写入规则

遵循 `system/memory-system.md` 的写入规则：

- **事实**（工具数据、用户原话）→ 直接写入 daily memory。
- **假设**（Agent 的临时推断）→ 写入 daily memory 并标注「暂时假设」。
- **洞察**（经本次循环确认的理解）→ 写入 daily memory，候选进入 long-term。
- **模式**（多次重复出现）→ 写入 long-term 并标注「待确认」，待用户确认后移入 profile。

#### 6.3 标记跟进事项

如果在本次循环中发现了需要后续关注的事项：

- 在 `memory/long-term.md` 中追加一条 `## 待跟进` 条目。
- 在下一次 CHECK 时优先读取该条目。

#### 6.4 更新游标

- 更新 `system/state/mail-cursor.json`（邮件去重游标，详见 `engine/email-protocol.md`）。
- 更新 `system/state/message-log.json`（消息日志）。
- 更新 `processed_mailids` 集合（裁剪超过 500 条时去掉最早的一半）。

#### 6.5 晚间类别判定（复盘时）

晚间块结束后的复盘阶段，按用户回复文本中的关键词自动判定每个晚间块归属类别，并写入当日 `memory/daily/{today}.md` 的「晚间」段，供次日 E1 统计覆盖快照（详见 `engine/cron-system.md` 的「晚间四类覆盖追踪」）。不需要用户手动分类。

| 类别 | 关键词（匹配时忽略大小写） | 示例活动 |
|------|--------------------------|---------|
| 精进 | 看、读、学、纪录片、课程、练习、技能、笔记、书 | 读《深度工作》、看纪录片、上线上课 |
| 休闲 | 刷剧、游戏、休息、放松、躺、随便、电影、综艺、音乐 | 看剧、打游戏、刷手机、发呆 |
| 锻炼 | 跑、健身、运动、走、跳、游泳、瑜伽、骑行、拉伸 | 跑步 5km、健身房、散步 |
| 副业 | 副业、项目、赛道、搞钱、研究、产品、方案、客户 | 研究 AI 赛道、搭副业网站 |

写入格式示例：

```markdown
## 晚间
- [块1] 20:00-21:00: 看纪录片《XXX》 -> 精进
- [块2] 21:00-22:00: 刷了两集剧 -> 休闲
- [块3] 22:00-23:00: 项目方案初稿 -> 副业
```

判定规则：

- 关键词匹配优先：命中某类别关键词即归入该类别；多个类别关键词同时命中时，按「精进 > 锻炼 > 副业 > 休闲」的优先级归入更具体的一类。
- 无法判定（用户未回复或文本不含任何关键词）时标记为「未分类」，不计入覆盖统计。

**输出**：Memory Update Record，包含：

- 写入的文件列表和内容摘要。
- 新增的待确认洞察。
- 更新的游标值。
- 标记的跟进事项。

**与 session-flow 的对应**：session-flow 的 Phase 6（闭环）是本步骤的消息通道版本。LEARN 的职责更广——不仅记录本次会话，还提取跨会话的模式、更新长期洞察、标记跟进事项。

---

## 四、与 session-flow 的对偶关系

algorithm 和 session-flow 是同一枚硬币的两面：

| 维度 | algorithm（本文件） | session-flow（engine/session-flow.md） |
|------|-------------------|--------------------------------------|
| 层级 | 推理框架（Agent 如何思考） | 状态机（消息何时收发） |
| 核心单元 | 6 步推理循环（CHECK→LEARN） | 6 阶段状态转移（采集→闭环） |
| 驱动方式 | 数据驱动：GAP 分析结果决定动作 | 事件驱动：Cron 触发、用户回复、超时 |
| 输出 | Action Plan、Verification Report、洞察 | 消息发送、会话上下文状态变更、归档 |
| 复用性 | 可用于邮件、IM、实时对话等多种通道 | 特化于邮件 + Cron 触达场景 |

**映射关系**：

```
algorithm 步骤          session-flow 阶段
─────────────────────   ─────────────────────
CHECK（了解现状）   ←→   Phase 1 采集 + Phase 4 理解的一部分
GAP（发现差距）     ←→   Phase 4 理解（分类 + 信号提取）
DECIDE（决定动作）  ←→   Phase 5 行动（Action 路由决策）
ACT（执行动作）     ←→   Phase 2 触达 + Phase 5 行动（执行）
VERIFY（工具验证）  ←→   Phase 6 闭环的写入验证
LEARN（记录洞察）   ←→   Phase 6 闭环的 memory 写入 + 游标更新
```

**关键区别**：session-flow 的 6 阶段是**顺序不可跳过**的（除了合法的提前退出路径）。algorithm 的 6 步是**逻辑上的完整推理周期**——在简单场景下可以压缩（如纯状态更新的 CHECK→GAP→DECIDE→ACT(记录)→VERIFY→LEARN 可以在一次推理中完成），但步骤本身不可省略。

---

## 五、与记忆系统的配合

algorithm 的每个步骤都与记忆系统（`system/memory-system.md`）有明确的读写关系：

| algorithm 步骤 | 读 memory | 写 memory |
|---------------|----------|----------|
| CHECK | `profile/user.md`、`memory/daily/*.md`、`memory/long-term.md`、`planning/*.md` | 不写入（仅读取） |
| GAP | `planning/*.md`（本期计划）、`reviews/*.md`（历史模式） | 不写入（仅分析） |
| DECIDE | `memory/long-term.md`（历史响应有效性） | 不写入（仅决策） |
| ACT | `profile/*.md`（执行前确认上下文） | `memory/daily/*.md`、`planning/*.md`、外部工具 |
| VERIFY | 读回刚写入的文件/API 状态 | 不写入（仅验证） |
| LEARN | `memory/long-term.md`（追加前检查已有条目） | `memory/daily/*.md`、`memory/long-term.md`、`system/state/*` |

**三不原则**（派生自 memory-system 的规则）：

1. 不把单次对话的推测写成长期事实。
2. 不在 CHECK 阶段写入任何内容。
3. 不在 VERIFY 未通过时标记任何已确认的完成状态。

---

## 六、与 ISA 系统的配合

ISA（Ideal State Artifact，见 `system/isa-system.md`）是 algorithm 中「GAP 分析」和「VERIFY 验证」的标准来源：

- **GAP 阶段**：ISA 的 Claims 定义了 Ideal State 的可验证标准。GAP 分析对照 ISA Claims 逐项判断 Current State 是否满足。
- **DECIDE 阶段**：如果当前任务有 ISA 文档，动作设计必须至少推进一条 Claim。如果任务没有 ISA 且复杂度需要，DECIDE 阶段标记 `ISA_NEEDED`。
- **VERIFY 阶段**：ISA Claims 的 `verification_method` 字段直接定义了验证方式。验证通过 = Claim 关闭。
- **LEARN 阶段**：如果某个 ISA Claim 反复未能关闭，这是值得记录的差距模式。

---

## 七、核心原则

### 7.1 Current → Ideal 爬山

每个动作都是朝 Ideal State 靠近的一步。如果无法说出这一步让什么更接近了理想状态，就不该执行这一步。

### 7.2 没有证据 = 没做完

声明只凭工具证据关闭。感觉、估计、假设都不算。

### 7.3 每个声明附带可证伪条件

说「已完成」的同时，说明「怎么验证」。声明如果无法被证伪（即无法设计验证方式），它就不是一个有效的完成声明——需要在 ISA 中重新定义。

### 7.4 倾听先于行动

（来自 Life Coach 共享教练流程）在 CHECK 和 GAP 阶段充分理解用户状态之前，不进入 DECIDE。在用户表达强情绪时，ACT 的优先级永远是情绪承接 > 任务推进。

### 7.5 最小下一步

（来自 ACT 承诺行动）DECIDE 输出的动作应该是「用户实际能做到的最小一步」，而不是理想的最大动作。行动设计要有同情心，也要现实。

### 7.6 一次一件事

（来自 Coaching Process 的「一次只问一个锋利问题」）单轮循环只聚焦一个 GAP 信号。如果多个差距同时存在，按优先级排序后只处理最优先的那个。不要让用户一次面对所有问题。

### 7.7 失败数据也有用

（来自 ACT 核心参考）如果用户未能按计划行动，不是「意志力差」——那是需要被理解的阻力数据。GAP 分析中的 `significant_deviation` 和 `emotional_blocker` 是信号，不是失败。

### 7.8 不要阻塞在工具失败上

（来自 memory-system 的「工具失败可降级」）如果外部工具（日历 API、滴答 MCP、天气 API）不可用，降级处理，不阻塞整个循环。先输出本地 Markdown 草案，待工具恢复后补录。

---

## 八、AI 工具实现指导

### 8.1 循环控制

algorithm 的 6 步是一个逻辑循环，但实现时不需要严格的 `while` 循环。在 Claude Code 环境中：

- 每次被触发时（Cron / 用户回复 / 手动），Agent 从 CHECK 开始执行完整 6 步。
- 如果是简单场景（如纯状态更新），6 步在一次推理中完成。
- 如果需要用户交互（如 coach 介入后等待用户回复），当前循环在 ACT（发送消息）后挂起，下一次收到用户回复时从 CHECK 重新开始。

### 8.2 步骤不可跳过但可压缩

6 步都必须执行，但在低复杂度场景下可以压缩：

- 纯状态更新：CHECK（读当日记录）→ GAP（自动判断为 `on_track`）→ DECIDE（`status_ack`）→ ACT（写入记录）→ VERIFY（读回确认）→ LEARN（写入摘要）。全程无需用户交互。
- 用户情绪信号：CHECK（读上下文）→ GAP（识别 `emotional_blocker`）→ DECIDE（`emotional_support`）→ ACT（发送情绪承接回复）→ VERIFY（确认发送成功）→ LEARN（记录情绪信号）。

### 8.3 与外部工具的接口

| 外部工具 | 在 algorithm 中的使用位置 | 接口约定 |
|---------|------------------------|---------|
| 滴答清单 MCP | CHECK（读待办）、ACT（创建/更新任务）、VERIFY（查询确认） | 见 `integrations/dida-mcp.md` |
| 企微邮件 API | ACT（发送）、VERIFY（确认发送） | 见 `engine/email-protocol.md` |
| 日历 API | CHECK（读日程） | 以本地计划文件为主要事实源，日历为辅助 |
| 天气 API | CHECK（早安节点） | 降级友好：不可用时跳过天气相关内容即可 |

### 8.4 调试与日志

每个步骤的执行应记录到日志（`system/state/message-log.json`）：

```json
{
  "session_id": "2026-07-29-0930-morning",
  "step": "GAP",
  "timestamp": "2026-07-29T09:30:05+08:00",
  "input_summary": "Current State: 今日计划 5 项，已完成 1 项，进度正常",
  "output_summary": "GAP: on_track, 无异常信号",
  "duration_ms": 120
}
```

## 文件完整性验证

### 6 步循环覆盖检查

```bash
grep -c "CHECK\|GAP\|DECIDE\|ACT\|VERIFY\|LEARN" system/algorithm.md
```

预期：至少匹配 6（六步各自出现）。

### 核心原则覆盖检查

```bash
grep -c "Current.*Ideal\|没有证据\|可证伪\|工具证据" system/algorithm.md
```

预期：至少匹配 3。

### 与 engine 层关联检查

```bash
grep -c "session-flow\|cron-system\|email-protocol" system/algorithm.md
```

预期：至少匹配 3。

## References

- `engine/session-flow.md`：6 阶段状态机，algorithm 的消息通道实例化。
- `engine/cron-system.md`：Cron 调度规则，algorithm 的触发源。
- `engine/email-protocol.md`：邮件收发协议，algorithm 在邮件通道下的 ACT 和 VERIFY 实现。
- `system/isa-system.md`：ISA 格式定义，algorithm 的 GAP 和 VERIFY 的标准来源。
- `system/memory-system.md`：记忆读写规则，algorithm 的 CHECK 和 LEARN 的读写依据。
- `integrations/dida-mcp.md`：滴答清单 MCP 集成，algorithm 的 CHECK 和 ACT 的外部工具。
- `coach/coaching-process.md`：教练流程，algorithm 中情绪承接的参考方法论。
- `coach/act-core.md`：ACT 底层操作参考，algorithm 中「最小下一步」和「失败数据也有用」的理论基础。
- `system/memory-system.md`：记忆读写规则，algorithm 中记忆读写规则的背景知识。
