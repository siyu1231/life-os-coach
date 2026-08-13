# 会话流状态机

本文件定义 Life Coach Agent 的邮件会话管理协议。它是一个 6 阶段状态机，描述一次完整的 AI 触达用户、接收回复、理解内容、执行行动、归档闭环的全链路。本文件是协议文档——描述 AI 工具应当做什么，而非规定具体代码实现。

## 核心概念

### 什么是「一次会话」

一次会话（Session）始于 Cron 系统触发一个时间点，终于该时间点的交互被归档闭环。关键约束：

- **一次会话只处理一封有效用户回复**（最先到达的那封）。如果用户在同一次会话中发送了多封邮件，仅第一封被纳入本次会话处理；后续邮件作为独立的新会话排队等待处理。
- **会话有生命周期**：从 Cron 触发开始计时，最长存活 30 分钟。超时后无论处于哪个阶段，强制进入闭环归档。
- **会话幂等**：同一封用户邮件不会被重复处理。通过 `mailid` 去重（参见 `engine/email-protocol.md` 的去重与幂等章节）。
- **不追问原则**：如果用户未回复，下一轮 Cron 触达不会追问「上次的问题你还没回答」。每轮触达独立成文（Agent 内部仍维护上下文）。

### 会话上下文

每个 Session 实例携带以下上下文数据（Session Context），贯穿 6 个阶段的决策：

| 字段 | 类型 | 来源 | 说明 |
|------|------|------|------|
| `session_id` | string | 系统生成 | 本次会话的唯一标识，格式 `YYYY-MM-DD-HHmm-{cron_slot}` |
| `cron_slot` | string | Cron 系统 | 触发本次会话的 Cron 时间点标签，如 `morning`、`mid_morning`、`night_review` |
| `channel` | string | 配置 | 消息通道类型：`email`、`im` |
| `day_type` | string | `getDayType()` | 当日日历类型：`workday`、`weekend`、`holiday` |
| `mode` | string | `determine_mode()` | 最终 Cron 模式：`workday`、`rest_day_default`、`pure_rest` |
| `phase` | enum | 状态机 | 当前所处阶段 |
| `phase_started_at` | timestamp | 系统时钟 | 进入当前阶段的时间戳 |
| `session_started_at` | timestamp | 系统时钟 | 会话创建时间戳（用于 30 分钟超时判断） |
| `last_sent_message_id` | string | 邮件 API | 最近一次 Agent 发出的消息 ID |
| `last_sent_timestamp` | timestamp | 系统时钟 | 最近一次发送的时间戳（用于 +5min/+10min 动态 Timer） |
| `user_message` | object | 邮件轮询 | 用户回复的邮件内容（`mailid`、`from`、`content`、`reply_to_mailid`、`send_time`） |
| `user_message_classification` | enum | Phase 4 | 用户回复的分类结果（见 Phase 4） |
| `action_taken` | string | Phase 5 | 记录本次会话采取的行动，供闭环写入 memory |
| `user_profile` | object | `profile/user.md` | 用户稳定底图（缓存读取） |
| `day_context` | object | 外部工具 | 当日日历摘要、待办列表、天气等实时上下文 |
| `followup_5min_timer_id` | string | 动态 Timer | +5min 跟进定时器 ID，用于用户提前回复时取消 |
| `followup_10min_timer_id` | string | 动态 Timer | +10min 归档定时器 ID |

## 6 阶段状态机总览

```
          ┌──────────────────────────────────────────────────────────┐
          │                                                          │
          ▼                                                          │
  ┌────────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    │
  │ 1.采集     │───▶│ 2.触达   │───▶│ 3.接收   │───▶│ 4.理解   │    │
  │ (Gather)   │    │ (Reach)  │    │ (Receive)│    │(Interpret)│    │
  └────────────┘    └──────────┘    └──────────┘    └──────────┘    │
       ▲                                     │             │         │
       │                                     │ 超时/无回复  │         │
       │                                     ▼             ▼         │
       │                               ┌──────────┐  ┌──────────┐   │
       └───────────────────────────────│ 6.闭环   │◀─│ 5.行动   │───┘
                                       │ (Close)  │  │ (Act)    │
                                       └──────────┘  └──────────┘
```

### 状态转换表

| 当前阶段 | 触发事件 | 下一阶段 | 说明 |
|---------|---------|---------|------|
| — | Cron 时间点到达 | 1. 采集 | 会话生命周期开始 |
| 1. 采集 | 上下文收集完成，判断应该发送 | 2. 触达 | 正常流程 |
| 1. 采集 | 判断为假期/跳过/静默 | 6. 闭环 | 无需触达，直接归档 |
| 1. 采集 | Step 4.5 发现未处理用户回复 | 4. 理解 | 优先处理用户回复，跳过本次主动发送 |
| 2. 触达 | 邮件发送成功，动态 Timer 已设置 | 3. 接收 | 等待用户回复 |
| 2. 触达 | 邮件发送失败（API 错误） | 6. 闭环 | 记录失败，归档 |
| 3. 接收 | 5min/10min 内收到用户回复 | 4. 理解 | 正常流程 |
| 3. 接收 | +10min 超时，用户未回复 | 6. 闭环 | 归档为`waiting_for_reply` |
| 3. 接收 | 收到特殊指令（暂停/恢复邮件等） | 6. 闭环 | 执行指令后立即闭环 |
| 3. 接收 | 会话总时长超过 30min | 6. 闭环 | 超时强制闭环 |
| 4. 理解 | 分类完成 | 5. 行动 | 按分类执行对应动作 |
| 5. 行动 | 行动执行完毕，无需回复用户 | 6. 闭环 | 写 memory 后归档 |
| 5. 行动 | 行动执行完毕，需要发送回复 | 2. 触达 | 回到触达发送回复邮件（同一会话内最多回一次） |

### 阶段不可跳过

6 个阶段按顺序推进，不可跳过。但「采集→闭环」（假期跳过）和「接收→闭环」（无回复超时）是合法的提前退出路径，此时采集和接收阶段各自完成了自己的职责（收集上下文 / 等待回复），其余阶段被标记为 `skipped`。

## Phase 1: 采集（Gather）

### 触发条件

Cron 系统的时间点到达（详见 `engine/cron-system.md` 的时间点定义表），包括两类：

- **Fixed Cron**：09:30、10:30、14:30、15:30、16:30、17:30、20:00、21:00、22:00、23:00 等固定时间点。
- **Dynamic Timer**：发送后 +5min 和 +10min 的一次性定时器触发。此时跳过采集的直接上下文收集，仅检查用户回复状态。

### 动作

采集阶段的职责是**收集本次触达所需的所有上下文数据**，不做发送决策之外的推理。

#### Step 1: 检查全局开关

```
if config.get("cron.enabled") is false:
    mark phase as "skipped_reason=global_disabled"
    transition to 6.闭环
```

#### Step 2: 判断日类型

调用 `getDayType(today)`（实现见 `engine/cron-system.md` 的工作日判断章节，含 timor.tech → mxnzp.com → 本地推算三级降级链路）。

获取 `day_type`：`workday` / `weekend` / `holiday` / `unknown`。

#### Step 3: 日间节点模式判定

根据 `mode`（最终模式，由 `getDayType()` 和 `commitments.md` 中 rest_day 声明共同裁决）决定节点行为。

| `mode` | Cron 槽位类型 | 行为 |
|--------|-------------|------|
| `workday` | 任意 | 正常执行，使用工作日消息模板 |
| `rest_day_default` | 休息日节点（10:00/14:30/17:30） | 正常执行，使用休息日默认消息模板（基于 commitments 提醒） |
| `rest_day_default` | 工作日专属节点（09:30/10:30/15:30/16:30） | 跳过：标记 `phase=skipped_reason=rest_day`，进入 6.闭环 |
| `pure_rest` | 任意休息日节点 | 正常执行，使用纯休息消息模板（仅日程事件，不追任务） |
| `pure_rest` | 工作日专属节点 | 跳过：标记 `phase=skipped_reason=pure_rest`，进入 6.闭环 |
| `workday` | 晚间节点（20:00/21:00/22:00） | 正常执行，使用晚间消息模板 |
| `rest_day_default` | 晚间节点（20:00/21:00/22:00） | 跳过：标记 `phase=skipped_reason=rest_day`，进入 6.闭环 |
| `pure_rest` | 晚间节点（20:00/21:00/22:00） | 跳过：标记 `phase=skipped_reason=pure_rest`，进入 6.闭环 |
| `workday`（mode 降级） | 任意 | 降级按工作日处理（宁可多触达不可漏触达），但记录 warning 日志 |

**模式判定在 Phase 1 Step 2 之后执行：**

```text
1. cal_type = getDayType(today)
2. user_rest = 读取 commitments.md：
   筛选：目标日期 == today AND 类型 == "rest_day" AND 状态 == "pending"
3. mode = determine_mode(cal_type, user_rest)
4. 将 mode 写入 session_context.mode
```

#### Step 4: 收集用户上下文

从以下位置读取数据，装入 `session_context`：

| 读取位置 | 内容 | 读取策略 |
|---------|------|---------|
| `profile/user.md` | 用户底图：作息、固定承诺、沟通偏好、精力线索、执行线索 | 会话级缓存：同一会话内只读一次 |
| `memory/long-term.md` | 跨文件综合洞察和索引 | 按需读取 |
| `memory/daily/{today}.md` | 当日已有记录 | 按需读取 |
| `memory/daily/{yesterday}.md` | 昨日状态和行动 | 按需读取（用于连贯性上下文） |
| `planning/commitments.md` | 跨天承诺追踪：未来意图、目标日期、前置检查 | 按需读取。**休息日节点**：额外筛选 rest_day 声明、目标日期 == "每日" 的 step、未来 7 天到期的里程碑 |
| 外部日历 API | 今日日程摘要 | 按需读取 |
| 外部待办 API | 今日待办概览 | 按需读取 |
| 天气 API | 当日天气 | 仅早安问候节点读取 |

读取策略：
- **早安问候（09:30）**：读取 `user.md`、`memory/long-term.md`、昨日和今日的 `memory/daily/*.md`、`commitments.md`、日历、待办、天气。
- **状态检查（10:30/15:30/16:30）**：读取 `user.md`、今日 `memory/daily/*.md`、`commitments.md`（检查当日到期的承诺）。
- **午后启动（14:30）**：读取 `user.md`、今日 `memory/daily/*.md`、日历（下午时段）、`commitments.md`（检查今日剩余承诺的前置条件）。
- **日终收尾（17:30）**：读取 `user.md`、今日 `memory/daily/*.md`、`commitments.md`（检查今日完成情况并标记过期的待确认）。
- **晚间复盘（23:00）**：读取 `user.md`、今日 `memory/daily/*.md`、`commitments.md`（休息日：轻量提示明日到期的承诺；工作日：检查明日承诺的前置条件）。
- **发送后跟进（+5min/+10min）**：不读取新文件，仅检查 `last_sent_message_id` 之后是否有新用户邮件。
- **休息日晨间启动（10:00）**：读取 `user.md`、昨日 `memory/daily/*.md`、`commitments.md`（筛选 rest_day 声明 + 今日到期 step + 每日 step + 未来 7 天里程碑）。不读取天气和日历（休息日不需要）。
- **休息日午后跟进（14:30）**：读取 `user.md`、今日 `memory/daily/*.md`、10:00 触达记录（如果有）。
- **休息日日终收尾（17:30）**：读取 `user.md`、今日 `memory/daily/*.md`、`commitments.md`（检查今日承诺执行状态）。
- **晚间第一块（E1，20:00）**：读取 `user.md`、今日 `memory/daily/*.md`、近 7 天 `memory/daily/*.md`（统计晚间四类覆盖缺口）、`commitments.md`（筛选四类标签的 pending step/goal + 长期目标晚间可执行步骤）、`planning/telos.md`（当前阶段可推进步骤）。不读取天气（晚间不需要）。
- **晚间第二块（E2，21:00）**：读取 `user.md`、今日 `memory/daily/*.md`（晚间第一块执行记录）、`commitments.md`（检查 E1 填充的默认事项是否被用户调整）。若用户未回 E1，检查 E1 发送后是否有新回复；仍无回复按 E1 填充默认继续。
- **晚间第三块（E3，22:00）**：读取 `user.md`、今日 `memory/daily/*.md`（晚间第一、二块执行记录）、`commitments.md`（检查前三块覆盖类别，优先补缺口）。

#### Step 4.5: 检查收件箱（Cron 触发时必做）

**设计原则**：Agent 是主动服务者，不是自说自话的广播器。在每次 Cron 主动触达用户之前，必须先检查用户是否有话说——如果用户已经回复了上一封邮件但 Agent 还没处理，就应该先处理用户的回复，再决定是否发送新的触达。

**检查时机**：Phase 1 完成上下文收集（Step 4）之后、发送决策（Step 5）之前。

**检查流程**：

```
1. 获取上一次 Agent 发送的邮件 ID（last_sent_message_id）
2. 调用 list_mail API 获取收件箱新邮件列表（详见 engine/email-protocol.md 的轮询流程）
3. 筛选条件（与 Phase 3 Step 2 完全一致）：
   - 发件人匹配用户邮箱
   - send_time 在 last_sent_timestamp 之后（时间窗口可能长达数小时）
   - 非系统邮件
   - mailid 不在去重集合中
4. 如果找到符合条件的未处理用户邮件：
   → 收集所有符合条件的邮件（上限 3 封，超过则取最近的 3 封，按时间倒序排列）
   → 按时间正序逐封处理（先发的先处理）
   → 取消本次 Cron 的主动发送
   → 跳转到 Phase 4（理解）+ Phase 5（行动）处理每封用户回复
   → 每封处理完成后写 memory；全部处理完成后进入 Phase 6（闭环）
   → 下一轮 Cron 时恢复正常的主动触达
5. 如果没有未处理邮件：
   → 继续 Phase 1 Step 5（发送决策）
```

**为什么上限 3 封**：
- Phase 1 Step 4.5 的时间窗口（可能数小时）比 Phase 3（5-10 分钟）大得多，用户可能在两次 Cron 之间写多封邮件。
- 如果用户在 09:30-10:30 之间连续发出 2 封邮件（如「今天状态不好」+「对了，顺便问一下明天的安排」），Agent 只处理一封会丢掉用户的完整诉求。
- 上限 3 封是安全阀——如果用户在一小时内狂发了 10 封邮件，Agent 不会在 10:30 的 Cron 槽位里陷入处理风暴。取最近 3 封（`list_mail` 按时间倒序返回，取前 3 封后按时间正序处理），超过 3 封的在下一轮 Cron 时继续处理。
- 邮件已由去重集合保护——已处理过的不会被重复处理，所以即使每轮 Cron 处理多封，也不会重复。

**Phase 3 保持「一封原则」**：
- Phase 3 的窗口只有 5-10 分钟，用户几乎不可能在这个窗口内连续发多封实质性邮件。
- 如果 Phase 3 轮询到超过 1 封新邮件（极其罕见），可能意味着用户快速回复了多个不同邮件——这种情况仍然只处理最新到达的那封，其余邮件会在下次 Cron 的 Phase 1 Step 4.5 中被捕获（因为它们标记为未处理）。

**为什么不是发起新会话**：
- 如果在 Cron 触发时发现未处理的用户邮件，不需要为它创建独立会话。
- 直接在当前 Cron 会话的框架内处理——复用已收集的上下文（profile、memory、planning），避免重复读取。
- 原 Cron 的发送意图被搁置——Agent 优先响应用户，而不是按预设时间表说话。

**与 Phase 3 的区别**：

| 维度 | Phase 3（Receive） | Phase 1 Step 4.5（Pre-send Check） |
|------|-------------------|-----------------------------------|
| 触发条件 | Agent 刚发送了一封邮件 | Cron 定时触发，Agent 还没有发新邮件 |
| 检查目的 | 用户回了刚才那封吗？ | 用户回了上一封吗？（可能几小时前发的） |
| 时间窗口 | 发送后的 5-10 分钟 | 从上次发送到现在（可能跨数小时） |
| 如果找到回复 | 进入 Phase 4 理解 + Phase 5 行动 | 跳过本次主动发送，直接进入 Phase 4 + Phase 5 |
| 如果没找到 | 5min 后再查一次，10min 归档 | 继续发送本次 Cron 的主动触达 |
| 重复检查保护 | mailid 去重集合 | mailid 去重集合（与 Phase 3 共享） |

**去重保护**：Step 4.5 和 Phase 3 使用同一个 `processed_mailids` 去重集合（见 engine/email-protocol.md 的去重与幂等）。如果一封邮件已经在 Step 4.5 被处理过，后续的 Phase 3 轮询不会再处理它。同样，Step 4.5 不会重复处理已经在之前会话中处理过的邮件。

#### Step 5: 发送决策

基于收集到的上下文，判断本次是否应该发送触达消息：

**应该发送的情况**（满足任一即发送）：

- 所有 Fixed Cron 时间点在对应规则下正常触发。
- 上一次发送后用户尚未回复，且当前时间点在上一次发送的 **60 分钟以上**（避免短时间内连续打扰）。

**不应该发送的情况**（满足任一即跳过）：

- 用户在上一封邮件中回复了「暂停一天」且今天尚未结束。
- 上一封邮件发送于 30 分钟之内（同一 Cron 槽位的重复触发，由 Cron 系统去重保证）。
- 用户在今天内已主动回复过至少一次（用户已参与对话），且当前时间点距离上一次 Cron 触达不足 60 分钟。

如果判断为「不发送」，标记 `phase=skipped_reason=no_send_decision`，进入 6.闭环。

### 出口

| 条件 | 下一阶段 |
|------|---------|
| 应该发送 | 2. 触达 |
| 假期跳过 / 全局禁用 / 不发送决策 | 6. 闭环（`skipped`） |

## Phase 2: 触达（Reach）

### 触发条件

Phase 1 判断「应该发送」后进入。

### 动作

#### Step 1: 生成消息内容

根据 `cron_slot`（时间点标签），从消息模板变量表中获取对应内容（邮件模板定义见 `engine/email-protocol.md` 的各时间点变量填充表）。

**消息生成约束：**

- 一封邮件一个问题原则（见 `engine/email-protocol.md` 的正文约束）。
- 纯文本内容不超过 200 字。
- 提供 2-3 个可选回复方向，降低用户回复门槛。
- 不追问历史未回复内容。
- 风格按 `cron.style` 参数切换（`warm` / `brief` / `challenge`）。

**生成的 Prompt 结构（供 AI 工具使用）：**

```text
你是 Life Coach Agent。当前时间点：{cron_slot}。
用户今日上下文：{day_context 摘要}。
用户底图关键信息：{user_profile 关键字段摘要}。

请按 {cron.style} 风格生成一封邮件：
- 主题：{TIME_LABEL}
- 正文：包含上下文简述、1 个核心问题、2-3 个可选回复方向。
- 约束：纯文本不超过 200 字，不追问历史，不给评价，不连续发送。
```

#### Step 2: 选择发送通道

根据当前配置的通道类型选择发送方式：

- **邮件通道**：调用企微邮件 API `compose_send`（详见 `engine/email-protocol.md` 的发送邮件 API）。
- **IM 通道**（未来扩展）：调用对应 IM 的消息发送 API。

#### Step 3: 发送并记录

```
// 伪代码
send_result = channel.send(
    to=user_profile.email,
    subject=TIME_LABEL,
    content=generated_html_content,
    content_type="html"
)

if send_result.errcode != 0:
    // 发送失败
    log_error("邮件发送失败", send_result)
    session_context.set("send_error", send_result)
    transition to 6.闭环  // 记录失败，归档
    return

// 发送成功
session_context.last_sent_message_id = send_result.message_id
session_context.last_sent_timestamp = now()
```

**发送失败处理：**

| 错误码类型 | 处理方式 |
|-----------|---------|
| `40001` / `42001`（token 过期） | 刷新 access_token 后重试一次。二次失败则记录错误并闭环。 |
| `84001`（邮件功能未开通） | 记录错误，明确告知用户需在企微管理后台开通邮箱权限。本次会话闭环。 |
| `84003`（收件人无效） | 记录错误，提示用户检查 `profile/user.md` 中的邮箱地址配置。本次会话闭环。 |
| 网络超时 | 重试一次（间隔 3 秒）。二次超时则记录错误并闭环。 |
| 其他错误 | 记录错误日志，闭环。不无限重试。 |

#### Step 4: 设置动态跟进 Timer

发送成功后，根据 `engine/cron-system.md` 的发送后动态 Timer 规则，设置两个一次性定时器：

```
// +5min 温和检查
if config.get("cron.followup_5min_enabled", true):
    timer_id = schedule_once(
        delay=5 * 60 * 1000,  // 5 分钟（毫秒）
        callback="check_user_reply_and_gentle_reminder",
        context={"session_id": session_context.session_id}
    )
    session_context.followup_5min_timer_id = timer_id

// +10min 归档等待
if config.get("cron.followup_10min_enabled", true):
    timer_id = schedule_once(
        delay=10 * 60 * 1000,  // 10 分钟（毫秒）
        callback="check_user_reply_and_archive",
        context={"session_id": session_context.session_id}
    )
    session_context.followup_10min_timer_id = timer_id
```

这两个 Timer 是动态的一次性定时器，而非 Fixed Cron。它们在 Phase 6（闭环）或用户提前回复时被取消。

#### Step 5: 日志记录

将本次发送记录写入消息日志（`system/state/message-log.json` 或等价存储），包含：`session_id`、`cron_slot`、`发送时间`、`message_id`、`通道类型`、`消息摘要`。

### 出口

| 条件 | 下一阶段 |
|------|---------|
| 发送成功 | 3. 接收 |
| 发送失败 | 6. 闭环 |

## Phase 3: 接收（Receive）

### 触发条件

Phase 2 发送成功后进入。此阶段的职责是**等待并获取用户回复**。

### 动作

#### Step 1: 启动邮件轮询

发送邮件后，立即将轮询频率切换为密集模式（详见 `engine/email-protocol.md` 的轮询策略）：

| 时段 | 轮询间隔 | 说明 |
|------|---------|------|
| 发送后 0-15 分钟 | 30 秒/次 | 密集轮询，快速感知用户回复 |
| 发送后 15-30 分钟 | 60 秒/次 | 降低频率，仍保持关注 |
| 会话超时（30 分钟后） | 停止轮询 | 进入 Phase 6 闭环 |

轮询实现参考 `engine/email-protocol.md` 的整体轮询流程（list_mail → get_mail_content → 去重 → 处理）。

#### Step 2: 过滤目标回复

轮询获取到新邮件后，按以下规则过滤：

**必须同时满足全部条件，才视为本次会话的有效用户回复：**

1. **发件人匹配**：`from` 字段匹配 `profile/user.md` 中配置的用户邮箱地址。
2. **时间窗口**：`send_time` 在 `last_sent_timestamp` 之后（即回复的是本次发送或之后的内容）。
3. **非系统邮件**：不是退信、自动回复、外出回复等系统邮件（通过 `subject` 中的关键词判断，如 `Undelivered`、`自动回复`、`Out of Office`）。
4. **未处理**：`mailid` 不在去重集合中（参见 `engine/email-protocol.md` 的去重与幂等）。

**一封有效回复原则**：当找到第一封满足上述全部条件的邮件后，立即停止过滤，该邮件即为本次会话的用户消息。

#### Step 3: 提取回复内容

对命中的用户邮件：
1. 调用 `get_mail_content` API 获取完整正文（`type=text` 以获取纯文本）。
2. 提取纯文本内容（strip HTML tags）。
3. 去除邮件客户端自动附加的签名、引用原文等噪音（通过 `--` 分隔线、`>` 引用前缀、`发件人:` 模式等特征识别）。
4. 将提取后的有效内容存入 `session_context.user_message`。

#### Step 4: 特殊指令检测

在纯文本内容中检测特殊指令（详见 `engine/email-protocol.md` 的「特殊指令集」）：

| 指令关键词 | 匹配方式 | 动作 |
|-----------|---------|------|
| `暂停邮件` / `停止邮件` | 包含匹配（不区分大小写） | 设置 `cron.enabled = false`，发送确认回复，本次会话闭环 |
| `恢复邮件` / `开启邮件` | 包含匹配 | 设置 `cron.enabled = true`，发送确认回复，本次会话闭环 |
| `暂停一天` / `今天暂停` | 包含匹配 | 设置今日暂停标记，发送确认回复，本次会话闭环 |
| `总结` / `本周总结` | 包含匹配 | 触发周总结生成（异步任务），通知用户正在生成，本次会话闭环 |

**特殊指令的优先级**：特殊指令检测在 Phase 4（理解）之前执行。如果命中特殊指令，直接执行对应动作并进入 6.闭环，不再进行 Phase 4 分类。

#### Step 5: 取消动态 Timer

如果用户已回复（包括特殊指令），立即取消 Phase 2 设置的 +5min 和 +10min 动态 Timer：

```
cancel_timer(session_context.followup_5min_timer_id)
cancel_timer(session_context.followup_10min_timer_id)
```

#### Step 6: 超时处理

**+5min Timer 触发时（用户未回复）：**

```
function check_user_reply_and_gentle_reminder(session_id):
    load session_context by session_id
    if session_context.user_message is not null:
        return  // 用户已回复，什么都不做
    if session_context.phase != 3:
        return  // 会话已不在接收阶段
    // 发送温和提醒
    send_gentle_message("刚才的消息看到了吗？不着急，想到了随时回。")
    session_context.last_sent_message_id = new_reminder_message_id
    session_context.last_sent_timestamp = now()
    // 记录日志：+5min 提醒已发送
    log("+5min 温和提醒已发送", session_id)
```

**+10min Timer 触发时（用户仍未回复）：**

```
function check_user_reply_and_archive(session_id):
    load session_context by session_id
    if session_context.user_message is not null:
        return  // 用户已回复，什么都不做
    if session_context.phase != 3:
        return
    // 用户未回复，归档等待
    session_context.action_taken = "waiting_for_reply"
    transition to 6.闭环(session_context)
```

**30 分钟超时**：如果会话自 `session_started_at` 起超过 30 分钟仍未收到用户回复，强制进入 6.闭环。此为兜底保护，避免会话无限挂起。

### 出口

| 条件 | 下一阶段 |
|------|---------|
| 收到有效用户回复（非特殊指令） | 4. 理解 |
| 收到特殊指令 | 6. 闭环（执行指令后） |
| +10min 超时无回复 | 6. 闭环（`waiting_for_reply`） |
| 30min 会话总超时 | 6. 闭环（`timeout`） |

## Phase 4: 理解（Interpret）

### 触发条件

Phase 3 收到有效用户回复（非特殊指令）后进入。

### 职责

理解阶段的职责是**分析用户回复的语义，将其归类为七种标准类型之一**，为 Phase 5（行动）提供决策依据。

### 分类体系

用户回复被分类为以下七种类型：

| 类型 | 标识 | 典型特征 | 示例 |
|------|------|---------|------|
| **纯状态更新** | `status_update` | 用户汇报进展、状态，无显式求助或深层情绪。语气中性或积极。 | 「下午推进了设计稿，基本完成。」「今天三件事都做完了。」 |
| **新建意图/计划** | `create_intent` | 用户表达了未来要做某事的意图——不是改现有的计划，而是创造一个新的计划/承诺/目标。有明确的目标日期或时间范围。 | 「我打算下个月开始备战公务员考试。」「周末想去看电影。」「下周开始每天跑步。」「今年结束前要完成那个项目。」 |
| **任务承诺** | `task_commitment` | 用户表达了近期要做的具体任务，语气如在滴答清单里记一条待办。有明确的时间指向（今天/明天/周末/下次）且可转化为单一动作。与 `create_intent` 的区别：`task_commitment` 是「记一个待办」，`create_intent` 是「立一个目标」。 | 「今天要交水电费」「明天记得取快递」「周末大扫除」「下次修一下门把手」「得给车做保养了」 |
| **情绪困难信号** | `emotional_signal` | 用户表达疲惫、挫败、焦虑、无力、迷茫等情绪。可能有具体困难，也可能只是「今天好累」的单纯情绪。 | 「下午效率很低，一直卡着。」「今天特别累，什么都不想干。」「感觉自己什么都做不好。」 |
| **修改请求** | `modification_request` | 用户对已有计划、优先级、节奏提出调整。语气中性，关注点在行动层面。修改的是**已有**计划，而非创建新计划。 | 「下午的优先级变了，先做 B 项目。」「明天能不能改成 8 点？」「暂停今天的提醒。」「之前说的考公推迟一个月。」 |
| **深度问题** | `deep_question` | 用户提出需要推理、分析、多步拆解的问题。涉及价值选择、复杂决策、长期方向、人际关系等。 | 「我在想是不是该换方向了。」「这个项目和另一个项目冲突了，不知道先做哪个。」 |
| **休息日声明** | `rest_day_declaration` | 用户表达了某天（或某段时间）全天外出/旅游/休息/请假的意图——不是在安排一个承诺，而是在声明某天不可用于工作/推进。 | 「周六去南京旅游」「下周三请假全天在医院」「周日想完全放空」 |

### 分类 Prompt（供 AI 工具使用）

```text
你是 Life Coach Agent 的分类器。分析以下用户消息，归类为七种类型之一。

用户消息内容：
'''
{user_message.content}
'''

用户上下文：
- 当前时间点：{cron_slot}（Agent 发送的邮件是 {TIME_LABEL}）
- 用户今日已记录状态：{今日 memory/daily 摘要}
- 用户底图关键信息：{user_profile 摘要}
- 已有承诺列表（用于判断是新承诺还是修改已有承诺）：{pending commitments 摘要}

分类标准：
1. 纯状态更新（status_update）：用户汇报进展、状态，语气中性或积极，无显式求助或深层情绪。
2. 新建意图/计划（create_intent）：用户表达了未来要做某事的明确意图，有时间、有行动、有目标日期，且这是新创造的计划/承诺——不是修改已有计划。
3. 任务承诺（task_commitment）：用户表达了近期要做的具体任务，语气如在滴答清单里记一条待办。有明确的时间指向（今天/明天/周末/下次）且可转化为单一动作。与 create_intent 的关键区别：task_commitment 是「记一个待办」（轻量、具体、近在眼前），create_intent 是「立一个目标」（有重量、较长周期、需拆解）。如果用户说「周末大扫除」→ task_commitment（具体动作，无需拆解）；如果用户说「下个月开始备战公务员考试」→ create_intent（长期目标，需要拆解和多步准备）。
4. 情绪困难信号（emotional_signal）：用户表达疲惫、挫败、焦虑、无力、迷茫等情绪，或使用负面评价自身能力的语言。
5. 修改请求（modification_request）：用户对已有计划、优先级、节奏、设置提出具体调整——调整的是已存在的东西，不是创造新的。
6. 深度问题（deep_question）：用户提出需要多步分析、涉及价值选择或复杂决策的问题。注意：如果用户是「不确定这事儿值不值得列入计划」的探索性问题，优先归为 deep_question 而非 create_intent。
7. 休息日声明（rest_day_declaration）：用户表达了某天（或某段时间）全天外出/旅游/休息/请假的意图——不是在安排一个承诺，而是在声明某天不可用于工作/推进。例如：「周六去南京旅游」「下周三请假全天在医院」「周日想完全放空」。

区分 create_intent vs modification_request 的关键：
- 如果用户说「我打算下周开始 ___」→ create_intent（新创造）
- 如果用户说「下周的那个事推迟到周三」→ modification_request（改已有）
- 如果用户说「我之前说想考公，现在想确认这个方向」→ create_intent（首次确认意图，此前只是讨论）
- 如果用户说「考公的计划我决定不做了」→ modification_request（cancel 已有承诺）

区分 create_intent vs task_commitment 的关键：
- 如果用户说「下个月开始备战公务员考试」→ create_intent（长期目标，需要拆解、分多步准备、需持续跟进）
- 如果用户说「今天要交水电费」→ task_commitment（轻量待办，单一动作，无拆解需要）
- 如果用户说「周末大扫除」→ task_commitment（具体动作，虽然「周末」是未来日期但无需拆解）
- 如果用户说「下次修一下门把手」→ task_commitment（单一 repair 动作，模糊时间「下次」）
- 如果用户说「得给车做保养了」→ task_commitment（具体动作，无明确日期但意图清晰）
- 判定标准：目标是否需要拆解为多个子任务或长期准备？是 → create_intent；否且可归为单一动作 → task_commitment
- 判定标准：用户语气像是在滴答清单里记一笔？是 → task_commitment；像是在表达一个目标/计划？是 → create_intent
- **部署实测补充（2026-08）**：**进行中任务**（「在做/正在做/接下来做/最后一个时间块在做 X」）也归 task_commitment，不是状态更新；「新增任务：X」「完成了一个新增任务 X」是强触发器（即使含「完成」字样也优先按新增任务处理）；状态更新仅限非任务类陈述（「电影看完了」「cron 异常已解决」），提到具体可执行任务（动宾结构）一律归 task_commitment；一句话含多个动宾结构时，每个动作分别建任务（见 dida-mcp.md「部署实测修正」第 2 条）

区分 create_intent vs rest_day_declaration 的关键：
- 如果用户说「周末要去看电影」→ create_intent（计划去做某件具体的事）
- 如果用户说「周六去南京旅游」→ rest_day_declaration（全天被占用，不是单一承诺）
- 如果用户说「周日想完全放空」→ rest_day_declaration（明确表达全天休息意图）
- 如果用户说「下周请假」→ rest_day_declaration（工作日请假，覆盖原工作模式）
- 边界：如果用户说「周末要去南京参加婚礼」→ 默认归为 create_intent + 额外标记该日为 rest_day？还是只归为 rest_day_declaration？优先归为 rest_day_declaration——全天外出意味着无法做其他事，不需要再问「还想做什么」。

输出格式（JSON）：
{
  "classification": "<七种类型之一>",
  "confidence": <0.0 到 1.0>,
  "summary": "<一句话总结用户说了什么>",
  "key_signals": ["<识别到的关键信号词或短语>"],
  "intent_fields": {
    "what": "<用户想做什么（create_intent）或声明了什么（rest_day_declaration）>",
    "when": "<目标日期/时间范围>",
    "type": "<commitment 类型：plan / goal / task / rest_day>",
    "prerequisite": "<用户提到的前置条件或检查项，如果有>"
  },
  "reasoning": "<归类依据，引用用户原文中的具体表述>"
}

注意：
- 分类时引用已有承诺列表来判断是否是新承诺 vs 修改已有承诺。
- 如果用户同时表达了状态和情绪，优先归为 emotional_signal（情绪优先原则）。
- 如果用户给的是明确的指令/请求（如「改成 8 点」），优先归为 modification_request（指令优先原则）。
- 如果用户提出了需要分析的问题，优先归为 deep_question（深度优先原则）。
- 如果用户表达了具体的未来意图（新创造的计划），优先归为 create_intent（意图优先原则）。
- 如果用户表达了近期具体的轻量待办任务，优先归为 task_commitment（任务承诺优先原则）。
- 如果无法确定，confidence 设为低于 0.6，并在 reasoning 中说明不确定性。
```

### 分类优先级规则

1. **情绪优先**：如果用户回复中同时包含状态更新和情绪表达（如「做完了但很累」），优先归为 `emotional_signal`。情绪需要先被承接，再谈其他。
2. **指令优先**：如果用户给的是明确的指令/请求（如「改成 8 点」），即使带有情绪（如「烦死了，改成 8 点吧」），优先归为 `modification_request`。用户给了清晰指令时，先执行再安抚。
3. **意图优先**：如果用户表达了未来要做某事的**具体意图**（有时间、有行动、有目标日期），且这是**创造一个新的未来承诺**而非**修改已有计划**，优先归为 `create_intent`。区分标准：「调整已有计划」→ `modification_request`；「创造新的未来承诺/项目/目标」→ `create_intent`。
4. **深度优先**：如果用户提出了需要分析的问题，即使描述中包含状态（如「今天我做了 A 和 B，但我在想是不是该换方向了」），优先归为 `deep_question`。用户在寻求帮助思考，而非仅仅汇报。
5. **休息声明优先**：如果用户表达了全天外出/休息/请假意图（含日期和时间范围），优先归为 `rest_day_declaration`，即使描述中包含情绪或状态内容。此类声明直接影响 Cron 调度模式，需要立即写入 commitments.md。
6. **任务承诺优先**：如果用户表达了近期具体的轻量待办任务（有明确时间指向且可转化为单一动作），优先归为 `task_commitment`。与 create_intent 冲突时，按「是否需要拆解」判断——不需要拆解 → task_commitment，需要拆解 → create_intent。

### task_commitment 匹配模式与自动推断

当分类为 `task_commitment` 时，从用户原文中提取时间信号，按以下规则自动推断 `due_date` 和 `priority`：

| 触发模式 | 匹配特征（关键词/句式） | 示例 | 推断 due_date | 推断 priority | 说明 |
|---------|----------------------|------|-------------|-------------|------|
| 「今天要 X」 | `今天` + 动作动词 | 「今天要交水电费」「今天必须把报销单填了」 | 当日日期 | `high` | 今日截止，默认高优 |
| 「明天记得 X」 | `明天` + 动作动词 | 「明天记得取快递」「明天记得带伞」 | 次日日期 | `medium` | 明日待办，默认中优 |
| 「周末 X」 | `周末` + 动作动词 | 「周末大扫除」「周末整理书架」 | 本周六日期 | `low` | 周末弹性时间，默认低优 |
| 「下次 X」 | `下次` + 动作动词 | 「下次修一下门把手」「下次记得备份」 | `null`（标记为 `someday`） | `low` | 无明确日期，进入「有空再做」池 |
| 「得 X 一下」 | `得`/`要` + 动作动词 + `一下` | 「得给车做保养了」「要理发了」 | `null`（标记为 `this_week`） | `medium` | 近期需做但无确切日期，默认本周 |

**自动推断规则：**

1. 如果用户原文中包含了明确的日期（如「8 月 5 号交报告」），优先使用用户指定的日期，不按模式推断。
2. 如果用户原文中包含了明确的优先级信号（如「很急」「不着急」「必须」），优先使用用户表达的优先级，不按默认值。
3. `due_date` 为 `null` 且标记为 `someday` 的任务，在滴答清单中不设截止日期，归入「收集箱」或等价清单。
4. `due_date` 为 `null` 且标记为 `this_week` 的任务，在滴答清单中设置截止日期为本周日。
5. 所有 `task_commitment` 的 priority/due_date 推断结果在 Action G 执行时写入对应字段，并在回复用户时予以确认。

### 分类结果的使用

分类结果写入 `session_context.user_message_classification`，供 Phase 5 路由到相应行动。同时：

- `summary` 字段写入 `session_context`，供后续 memory 写入使用。
- `key_signals` 字段用于判断是否需要更新 `profile/user.md` 中的精力/执行线索（如多次出现「下午效率低」可能意味着精力低谷模式）。
- `intent_fields` 字段（`create_intent` 和 `rest_day_declaration` 分类有值）写入 `session_context.create_intent`，供 Phase 5 的 Action E 使用。
- `confidence` 低于 0.6 时，Phase 5 应采取更保守的行动（优先询问澄清而非直接执行）。

### 出口

分类完成后进入 Phase 5（行动）。无论分类结果如何，此阶段不可跳过。

## Phase 5: 行动（Act）

### 触发条件

Phase 4 分类完成后进入。

### 职责

行动阶段的职责是**根据用户回复的七种分类，分别执行对应的 Action 策略**。`rest_day_declaration` 分类由 Action E 第 5 子项承接（写入 commitments.md 并切换 Cron 模式），不需要独立 Action 策略。每个策略包含三个核心要素：承接方式、操作动作、是否回复用户。

### 七种分类的 Action 策略

#### Action A: 纯状态更新（status_update）

**目标**：确认收到，正向反馈，记录数据。

**承接方式**：
- 简短肯定用户的状态/进展（如果用户汇报了完成的进展，给予正向反馈）。
- 不追问、不分析、不给建议。

**操作动作**：
1. 将用户汇报的状态写入 `memory/daily/{today}.md`（追加到当日记录中）。
2. 如果用户汇报了完成事项，更新 `planning/daily-plan.md` 的实际完成状态。
3. 检查用户的描述中是否有新的精力/执行线索（如「下午效率高」「早上起晚了」），如果与已知模式不同，标记为候选洞察，待下次复盘时确认。

**是否回复用户**：只在该 Cron 槽位自身的回复模板之外，额外附加一句简短的确认（如「收到，记录下了。」）。如果当前已是本轮会话的最后阶段（如日终收尾），不追加额外回复。

**回复邮件模板**：

```text
收到，记录下了。
{CURRENT_TIME_LABEL 的默认消息}
```

#### Action B: 情绪困难信号（emotional_signal）

**目标**：承接情绪，降低羞耻，不急于解决。

**承接方式**：
- 先映照情绪，而非直接给解决方案。使用「接住情绪与映照处境」的方法（参考 `coach/coaching-process.md` 的第二步）。
- 正常化用户的感受（如「下午效率低是精力曲线的自然波动」）。
- 不评价（不说「你今天只完成了这个？」）。
- 如果用户表达了较强的疲惫/挫败，主动降低期望：「今天不用再推进什么，先照顾好自己。」
- 如果用户表达了疑似需要专业支持的内容（持续强烈抑郁、自伤等），按 coaching-process.md 的「边界与转介」原则处理。

**操作动作**：
1. 将用户的情绪信号和触发场景写入 `memory/daily/{today}.md`。
2. 检查该情绪信号是否与已知模式重复（查阅 `memory/long-term.md` 和近期的 `memory/daily/*.md`）：
   - 如果首次出现：标记为「待观察」。
   - 如果近期重复出现（如连续 3 天下午效率低、疲惫）：标记为「模式候选」，建议在下次复盘时与用户确认。
3. 如果情绪信号涉及精力/执行线索的模式，写入候选洞察到 `memory/long-term.md`（标注为「待确认」）。
4. 不使用 `planning/daily-plan.md` 中的任务推进逻辑（不催促进度）。

**是否回复用户**：必须回复。回复内容以情绪承接为主，包含：
- 映照情绪（1-2 句，引用用户的具体表达）。
- 正常化（1 句，降低羞耻）。
- 轻量建议或邀请（1 句，可选，如「需要聊聊还是先自己待一会？」）。

**回复邮件模板**：

```text
{映照情绪}。这很正常，{正常化理由}。
{CURRENT_TIME_LABEL 的默认消息，可酌情缩短或替换为关怀内容}
```

#### Action C: 修改请求（modification_request）

**目标**：确认修改，快速执行，减少摩擦。

**承接方式**：
- 直接确认理解用户的修改要求。
- 执行修改，并告知修改结果。
- 如果修改的内容不明确（如「优先级变了」但未说明新优先级），发送一个简短确认问题。

**操作动作**：
1. 根据修改的类型执行对应操作：

| 修改类型 | 操作 |
|---------|------|
| 计划/优先级修改 | 更新 `planning/daily-plan.md` 或 `planning/weekly-plan.md` 中的对应条目 |
| Cron 配置修改（如时间调整） | 更新 `profile/user.md` 的 `## Cron配置` 章节中的对应参数 |
| 消息偏好修改 | 更新 `profile/user.md` 的沟通偏好部分 |
| 未来承诺修改（延期/取消/改日期） | 更新 `planning/commitments.md` 中对应条目的状态或日期字段 |
| 未完成项处置（顺延/取消/拆小/降级，针对日任务） | 更新 `planning/carryover.md` 中对应条目的处置状态与顺延计数，并同步滴答（顺延 → update_task dueDate；取消 → delete_task；拆小 → 按回复拆分新任务；降级 → update priority） |
| 其他设置修改 | 更新对应配置文件 |

2. 将修改记录写入 `memory/daily/{today}.md`。
3. 如果修改影响了今日后续的 Cron 行为（如暂停一天），设置对应标记供后续 Cron 触发时检查。
4. **告知后续影响**：除了告知已执行的修改本身，还告知用户这次修改带来的连锁影响：
   - 如果修改了优先级：哪个任务被推后了？
   - 如果修改了日期：Agent 下次什么时候会提到这个承诺？
   - 如果取消了计划：该计划关联的 ISA 或承诺是否需要更新？
5. **标记计划已变更**：在 Phase 6 写入 `memory/daily/{today}.md` 时，追加一条标记 `⚠️ plan-changed: {修改类型} — {修改内容}`。该标记在下一次 Cron 触发的 CHECK 阶段被读取，作为「上次发生了什么变化」的入口数据。

**是否回复用户**：必须回复。回复内容为：
- 确认已理解和执行修改。
- 告知具体修改了什么。
- 如果修改不明确，附加一个简短的确认问题。

**回复邮件模板**：

```text
好的，已{修改的具体内容}。
{如果 Cron 行为有变化，说明后续影响}
{CURRENT_TIME_LABEL 的默认消息，如果相关}
```

#### Action D: 深度问题（deep_question）

**目标**：理解问题，给予有质量的引导或分析，但不在邮件里做完整教练会话。

**承接方式**：
- 确认收到问题，表示正在认真对待（不急于给答案）。
- 如果问题可以在邮件中以简洁方式回应（2-3 个思考方向），在回复中给出。
- 如果问题复杂，不适合邮件异步处理，引导用户进入实时对话或在下一个时间点多分配时间讨论。

**操作动作**：
1. 将用户的问题和关键上下文记录到 `memory/daily/{today}.md`。
2. 检索相关参考资源（`coach/skills/` 下的对应 skill 的 cases 和 toolkit），获取类似问题的处理框架。
3. 根据问题复杂度判断：
   - **轻量问题**（可在 200 字内给出有意义的引导）：在回复邮件中直接给出 2-3 个思考方向或一个可选框架。
   - **中等问题**（需要稍多展开）：在回复中给出 1 个核心思路，邀请用户在下一个时间点深入讨论。
   - **复杂问题**（需要对话式教练过程）：在回复中确认收到，表示「这个问题值得认真聊聊」，约定在下一个时间点（或用户方便时）集中讨论。

4. 如果问题涉及已有 skill 的领域（如 `procrastination-execution`、`life-vision`、`complex-problem-solving`），在回复中声明将调用对应的方法论。

**是否回复用户**：必须回复。回复内容包含：
- 映照问题的核心（1 句）。
- 1-3 个思考方向或一个分析框架（简洁版）。
- 如果需要深入讨论，明确建议时间和方式。

**回复邮件模板**：

```text
收到，这是个好问题。{映照问题核心的一句话}。

几个思考方向：
1. {方向 1}
2. {方向 2}
（如合适，可以有第 3 个方向）

{CURRENT_TIME_LABEL 的默认消息，或替换为「如果愿意，可以在下一个时间点多聊几句。」}
```

#### Action E: 新建意图/计划（create_intent）

**目标**：将用户表达的未来意图结构化存储，确认关键字段，确保 Cron 系统在未来时间点主动跟进。

**承接方式**：
- 先确认关键字段——不急着保存，先确保理解正确。
- 区分「清晰意图」和「模糊意图」：
  - 清晰意图（有具体时间、有具体行动）→ 直接存入 `planning/commitments.md`
  - 模糊意图（「我想做 ___ 但还不确定时间/方式」）→ 存入 `memory/long-term.md` 的「待确认意图」区域，下一次对话时确认

**操作动作**：

1. 如果用户提供的意图信息足够清晰（同时满足：知道做什么 + 大致知道何时做）：
   - 将意图写入 `planning/commitments.md` 的当前表，生成 #ID。
   - 填入字段：承诺内容（引用用户原话）、目标日期、来源对话、提醒时间（根据目标日期自动判断——在同一天 → `all`；在数天内 → `night_review`；在数周后 → `morning`）。
   - 如果用户提到了前置条件（如「看电影前先买票」），填入前置检查列。

2. 如果用户的意图信息不够清晰（缺少日期、缺少具体行动或范围过广）：
   - 在回复中追问 1-2 个关键问题上限（一封邮件不超过两个问题）。
   - 在本次 Phase 6 写入 `memory/daily/{today}.md` 时标记一条「待确认意图」。
   - 下次 Cron 触达时，Agent 读取 `memory/long-term.md` 的「待确认意图」部分，如果未过期，在合适的 Cron 节点提醒确认。

3. 如果意图涉及一个需要长期准备的事项（如「下个月开始备考」）：
   - 除了写入 `planning/commitments.md`，还要检查是否需要创建 ISA 文档（如果该意图足够大，满足「大于一眼能回答」的标准——见 `system/isa-system.md`）。
   - 如果意图涉及滴答清单（如「下周开始每天跑步」是日常任务），检查是否需要创建重复任务。

4. 如果意图涉及**需要提前做什么**（如周末看电影需要提前买票）：
   - 将前置动作作为一条单独的 commitment 写入——与主承诺关联（`前置动作: 为 #3 买电影票`）。

5. 若用户的意图类型为 `rest_day`（全天休息声明）：
   - 写入 `planning/commitments.md`，类型设为 `rest_day`
   - 承诺内容填用户声明摘要（如「南京旅游（全天外出）」）
   - 父承诺、前置检查、提醒时间留空（填 `-`）
   - 若用户声明了日期范围（如「初一到初五在老家」）→ 每个日期创建一条独立记录（便于每日 CHECK 命中）
   - 回复用户确认：「收到，{日期} 标记为休息日了。那天我不会追问任务，但如果有日程事件（如高铁时间）会照常提醒。」

**写入数据**：

| 写入位置 | 内容 | 何时写入 |
|---------|------|---------|
| `planning/commitments.md` | 待办承诺（一条新行） | ACT 阶段直接写入（Tier A — 用户刚确认的意图） |
| `planning/commitments.md` | 待办承诺（rest_day 类型——日期范围） | 每个独立日期一条记录 |
| `memory/daily/{today}.md` | 「用户表达了 ___ 的意图，已在 commitments.md 记录」 | Phase 6 闭环时记录 |
| `memory/long-term.md` →「待确认意图」区域 | 不清晰的意图，待下次对话确认 | Phase 6 闭环时标记 |
| ISA 文档（可选） | 如果意图足够大，需要 ISA | ACT 阶段创建 |

**是否回复用户**：必须回复。回复内容包含：

```
好的，记下了。{意图的一句话确认}。

→ 目标日期：{YYYY-MM-DD}
→ 在此之前：{前置检查项，如果有}
→ 我会在接近那天时提醒你。

（如果信息不清晰：追问 1-2 个问题）

{CURRENT_TIME_LABEL 的默认消息}
```

**回复邮件模板**：

```text
收到，已经把这条放进承诺追踪里了。

- 📌 {意图的一句话确认}
- 📅 目标日期：{YYYY-MM-DD}
- ⏰ 提醒时机：我会在 {提醒提前量} 列出来
- 🔔 前置：{前置检查项，如果没有就写"无"}

{如果意图不清晰，追问 1-2 个问题}
```

#### Action F: 休息日声明（rest_day_declaration）

由 Action E 第 5 子项承接（见上文 Action E 的操作动作第 5 项）。

#### Action G: 任务承诺（task_commitment）

**目标**：将用户近期轻量任务承诺转化为滴答清单待办，幂等检查避免重复。遵循「不双写原则」（dida-mcp.md）：MCP 可用时滴答是状态的事实源，MCP 不可用时降级到本地文件。

**承接方式**：
- 快速确认任务和时间——不展开、不拆解、不追问「要不要做成目标」。
- 用户表达的是轻量待办，承接方式也应轻量——一句话确认，立刻执行。

**操作动作**：

**路径 A：滴答 MCP 可用（正常路径）**

1. **幂等检查（`get_today_tasks`）**：
   - 调用 `get_today_tasks` 获取今日已有待办列表。
   - 搜索关键词从 `task_commitment` 识别出的「核心名词 + 核心动词」中提取（如「交水电费」→ 搜索「水电费」）。
   - 比较已存在任务的标题和用户原文的相似度（模糊匹配，允许措辞差异）。
   - 如果找到相似度 > 70% 的已有任务：不创建新任务，回复用户「这条已经记过了哦。{已有任务标题}」。跳过后续创建步骤。
   - 如果未找到相似任务：继续执行创建步骤。

2. **创建滴答清单任务（`create_task`）**：
   - 调用 `create_task` 将 task_commitment 写入滴答清单。
   - 参数：`title`（用户原文核心表述）、`due_date`（推断值或 null）、`priority`（推断值）。
   - 如果 `due_date` 为 `null` 且标记为 `someday`：不传 `due_date`，任务归入收集箱。
   - 如果 `due_date` 为 `null` 且标记为 `this_week`：传本周日日期。
   - 遵循「不双写原则」：MCP 可用时不将任务副本写入本地 `planning/commitments.md`（滴答是事实源）。

3. **回复用户确认**：
   - 必须回复，告知用户已创建滴答清单任务。
   - 确认内容包含：任务名称、截止日期/清单位置、优先级。

**路径 B：滴答 MCP 不可用（降级路径）**

1. **降级写入本地**：
   - 将 task_commitment 作为一条记录追加到 `planning/commitments.md`。
   - 类型设为 `task`（与 `plan`、`goal`、`rest_day` 区分）。
   - 填入字段：承诺内容（引用用户原话）、目标日期（即推断的 due_date）、来源对话、优先级、提醒时间留空（填 `-`）。
   - 在记录末尾标注 `[待同步到滴答]`。

2. **回复用户确认**：
   - 必须回复，告知用户「滴答清单暂时不可用，已帮你记在本地，连接恢复后会同步过去。」
   - 确认内容包含：任务名称、截止日期/清单位置、优先级。

**是否回复用户**：必须回复。

**回复邮件模板**：

```text
收到，已记入滴答清单。

- ✅ {任务一句话确认}
- 📅 {截止日期，如"今天"、"明天"、"本周六"、"无截止日期（收集箱）"}
- 🔔 优先级：{高/中/低}

{CURRENT_TIME_LABEL 的默认消息，如果相关}
```

**降级回复模板（滴答不可用时）**：

```text
滴答清单暂时不可用，已帮你记在本地。

- ✅ {任务一句话确认}
- 📅 {截止日期，如"今天"、"明天"、"本周六"、"无截止日期（收集箱）"}
- 🔔 优先级：{高/中/低}
- ⚠️ 连接恢复后会自动同步到滴答清单，不用担心丢失。

{CURRENT_TIME_LABEL 的默认消息，如果相关}
```

**Action G 补充：完成报告处理（部署实测 2026-08）**

用户回复「已经完成了 X」「X 已完成」时：先 `search_task`/`get_task_by_id` 找滴答对应任务——找到 → `complete_task(project_id, task_id)` 勾掉（验证返回 `status=2` + `completedTime`）；**搜不到对应任务（如「完成了一个新增任务 X」）→ 不得静默跳过**——说明 X 从未入滴答，先按 task_commitment 补建（幂等查重），再按完成报告勾掉。勾选结果与 task_id 记录到当日 daily。同时注意：用户最新回复可推翻此前槽位记录的完成状态（详见 cron-system.md 回顾规则第 6 条）。

### Action 执行后的判断

每个 Action 执行完毕后，判断是否需要回到 Phase 2（触达）发送回复邮件：

| Action 类型 | 需要回复用户？ | 下一阶段 |
|------------|-------------|---------|
| A: status_update | 否（仅附在 Cron 消息中） | 6. 闭环 |
| B: emotional_signal | 是 | 2. 触达（发送回复后 → 3.接收） |
| C: modification_request | 是 | 2. 触达（发送回复后 → 3.接收） |
| D: deep_question | 是 | 2. 触达（发送回复后 → 3.接收） |
| E: create_intent | 是 | 2. 触达（发送回复后 → 3.接收） |
| F: rest_day_declaration | 是（见 Action E 子项 5） | 2. 触达（发送回复后 → 3.接收） |
| G: task_commitment | 是 | 2. 触达（发送回复后 → 3.接收） |

**注意**：回到 Phase 2（触达）发送回复后，会话会再次进入 Phase 3（接收）等待用户二次回复。但会话生命周期不变（仍从首次 `session_started_at` 算起），且同一会话内**最多允许一次回环**（即发送回复后如果用户二次回复，优先开启新会话处理）。

### 出口

| 条件 | 下一阶段 |
|------|---------|
| 无需回复用户（Action A） | 6. 闭环 |
| 需要回复且已发送（Action B/C/D/E/F/G） | 3. 接收（设置一次后续回复处理机会）或 6. 闭环（如果在 30min 超时窗口内） |

## Phase 6: 闭环（Close）

### 触发条件

以下任一条件满足时进入闭环：

- Phase 1 判断不发送（假期跳过、全局禁用、发送决策为否）。
- Phase 2 发送失败。
- Phase 3 用户回复了特殊指令。
- Phase 3 超时无回复（+10min 或 30min）。
- Phase 5 行动执行完毕且无需继续对话。
- 会话总时长超过 30 分钟（兜底保护）。

### 职责

闭环阶段的职责是**清理会话资源、持久化记录、更新游标、释放定时器**。

### 动作

#### Step 1: 取消所有动态 Timer

```
cancel_timer(session_context.followup_5min_timer_id)
cancel_timer(session_context.followup_10min_timer_id)
```

确保没有遗留的定时器在会话结束后意外触发。

#### Step 2: 写入 Memory

将本次会话的关键信息写入记忆系统（详见 `system/memory-system.md`）：

**写入 `memory/daily/{today}.md`：**

```markdown
## {cron_slot} 触达记录（{session_id}）

- 时间：{timestamp}
- 会话结果：{会话结果的简短描述}
- 用户回复分类：{classification 或 "未回复"}
- 用户关键表述：{user_message.summary 或 "无"}
- Agent 执行动作：{action_taken}
- 待观察线索：{如果 Phase 5 标记了候选洞察，在此列出}
```

**写入 `system/state/mail-cursor.json`（邮件去重游标）：**

更新 `last_processed_mailid` 和 `last_poll_time`。将本次处理的 `mailid`（如有）追加到 `processed_mailids` 集合中。集合超过 500 条时裁剪最早的一半。

#### Step 3: 更新消息日志

将本次会话的完整摘要写入 `system/state/message-log.json`（或等价存储），包含：`session_id`、`cron_slot`、`开始时间`、`结束时间`、`各阶段耗时`、`用户是否回复`、`分类结果`、`行动摘要`。

#### Step 4: 检查是否需要更新用户画像

如果在本次会话中发现了新的稳定线索（如 Phase 5 标记的精力/执行线索候选），且该线索在前几次会话中也出现过（满足「多次出现」条件），则：

1. 在 `memory/long-term.md` 中追加一条「待确认」洞察。
2. 不直接写入 `profile/user.md`（用户稳定底图的写入需用户确认，参见 `system/memory-system.md` 的写入规则）。

#### Step 5: 释放会话资源

```
delete session_context  // 或标记为 archived
update session registry: status = "closed"
log("会话闭环完成", session_id)
```

#### Step 6: 恢复常规轮询频率

如果之前在 Phase 3 中将邮件轮询切换为密集模式（30 秒/次），恢复为常规待机频率（5 分钟/次或 10 分钟/次，取决于时段）。

### 出口

闭环是终止状态，无下一阶段。系统回到 IDLE 状态，等待下一个 Cron 触发启动新的会话。

## 边界情况处理

### 无回复

**场景**：Agent 发送邮件后，用户在 +5min 和 +10min Timer 触发前均未回复。

**处理**：
1. +5min：发送温和提醒（如果 `cron.followup_5min_enabled` 为 `true`）。
2. +10min：不再发送消息，标记本次会话为 `waiting_for_reply`，进入 Phase 6 闭环。
3. 今日后续 Cron 节点：正常触发，但每封触达邮件的内容不追问上一封未回复的邮件。
4. 如果用户连续 3 个 Cron 时间点未回复（即一天的日间节点全部未回复），当天剩余的日间节点自动静默（不再发送），仅保留晚间 23:00 复盘。

**不对用户做负面归因**：不假设用户「不重视」「忘了」「故意不回」。Agent 内部只记录 `waiting_for_reply`，不附加任何评价性标签。

### 邮件发送失败

**场景**：Phase 2 调用邮件 API 返回错误。

**处理**：
1. 按错误码分类处理（见 Phase 2 的发送失败处理表）。
2. Token 过期类错误重试一次；配置类错误（如邮箱未开通）记录并通知用户；网络超时重试一次。
3. 本次会话标记为 `send_failed`，进入 Phase 6 闭环。
4. 如果连续 3 个 Cron 时间点均发送失败（累计），暂停后续 Cron 触达，记录 ERROR 级别日志，并尝试通过备用通道（如 IM）通知用户「邮件触达异常」。

### 非工作日模式切换

**场景**：Cron 触发日间节点时，`mode`（最终模式）非 `workday`。

**处理**：
1. Phase 1 判定为 `rest_day_default` 或 `pure_rest` 时，工作日专属节点跳过（标记 `skipped_reason=rest_day` 或 `skipped_reason=pure_rest`），共享节点照常执行。
2. 休息日节点（10:00/14:30/17:30）根据 mode 选择对应模板。
3. 晚间 23:00 节点照常执行（三种模式的消息模板不同）。
4. 连续假日抑制规则见 `engine/cron-system.md` 的「假期连续提醒抑制」章节。

### API 降级时的工作日判断不确定

**场景**：`getDayType()` 返回 `unknown`（两个外部 API 均不可用，本地推算因系统时间异常而无法执行）。

**处理**：
1. 降级按工作日处理（宁可多触达不可漏触达）。
2. 记录 WARNING 级别日志：「工作日判断 API 全部不可用，降级按工作日处理」。
3. 在下一次成功调用 API 后，检查降级期间是否误在节假日发送了日间消息。如果有，在随后的晚间复盘消息中附加一句简短致歉：「昨天可能打扰到你了，如果需要调整节奏，随时告诉我。」

### 用户短时间内多次回复

**场景**：Phase 3 找到第一封有效回复进入 Phase 4 后，用户又发送了第二封邮件。

**处理**：
1. 本次会话只处理第一封（已在 Phase 3 中锁定）。
2. 第二封邮件不会丢失——下次轮询时，`mailid` 不在去重集合中，且其 `send_time` 在 `last_poll_time` 之后，会被下次轮询获取。
3. 第二封邮件作为独立的新会话处理（从 Phase 1 开始，但采集阶段中的发送决策可能因距离上次触达不足 60 分钟而跳过）。

### 用户在 Agent 回复后再次回复

**场景**：Phase 5 执行了 Action B/C/D，生成了回复邮件发送给用户，用户再次回复。

**处理**：
1. 同一会话在收到用户二次回复后，优先开启新会话处理（因为本次会话已接近生命周期限制）。
2. 如果距离本次会话结束还有充足时间（>10min），可以在当前会话内处理，但不再回环（即收到二次回复后执行 Action 但不再次发送消息）。

### 并发会话

**场景**：两个 Fixed Cron 时间点之间间隔为 1 小时（如 09:30 和 10:30），但动态会话（包含 +5min/+10min Timer）可能跨越到下一个 Fixed Cron 触发时间。

**处理**：
1. 每个 Cron 触发创建一个独立会话，使用独立的 `session_id`。
2. 同一时间最多存在一个活跃会话（当前正在等待用户回复的会话）。
3. 如果新的 Cron 触发时上一个会话仍在 Phase 3（等待回复），上一个会话被标记为 `superseded`（被取代）并强制闭环。
4. 被取代的会话在闭环时记录 `superseded_by={new_session_id}`。

### 长期未交互

**场景**：用户连续多天未回复任何邮件（如休假、出差、遗忘）。

**处理**：
1. 第 1 天：所有时间点正常发送。
2. 第 2 天：仅发送早安问候和晚间复盘，跳过日间状态检查节点。
3. 第 3 天起：仅发送晚间复盘（23:00），日间全部静默。
4. 用户任何时候回复后，立即恢复完整日间调度。
5. 此行为由 `cron-system.md` 中的「假期连续提醒抑制」逻辑驱动（长期未交互与连续假期的处理策略类似）。

## 与 engine 层其他模块的关系

- **cron-system.md**：session-flow 的调度上游。每个 Cron 时间点触发时，session-flow 调用 `getDayType()` 判断日类型，读取 Cron 配置参数决定是否触达，发送后设置 +5min/+10min 动态 Timer。
- **email-protocol.md**：session-flow 的消息通道。Phase 2 通过 email-protocol 的模板变量表生成消息内容和主题，调用 `compose_send` API 发送；Phase 3 通过 email-protocol 的轮询策略（list_mail → get_mail_content）接收用户回复。

## 与 system 层模块的关系

- **system/algorithm.md**：定义了 Agent 的执行循环和推理框架。session-flow 的状态机是 algorithm 中「执行循环」在邮件场景下的具体实例化。Phase 4（理解）的分类 Prompt 和 Phase 5（行动）的五种 Action 策略遵循 algorithm 中定义的推理-行动范式。
- **system/memory-system.md**：定义了记忆的读写规则和维护周期。Phase 1（采集）依据 memory-system 的「启动时读取」规则决定读取哪些文件；Phase 6（闭环）依据 memory-system 的「写入规则」决定写入哪些内容到 daily memory、long-term memory、cursor 和 user profile。

## AI 工具实现指导

### 通用实现要点

1. **状态机引擎**：推荐使用有限状态机（FSM）库或简单的 switch-case 实现。每个阶段对应一个 handler 函数，阶段之间的转换通过 `session_context.phase` 字段控制。
2. **会话隔离**：每个 `session_id` 对应独立的 `session_context` 对象。推荐使用内存中的 Map/Dictionary 存储活跃会话，session 闭环后持久化摘要并释放内存。
3. **超时机制**：使用语言/框架的 Timer 或 Scheduler 实现会话 30 分钟超时和 +5min/+10min 动态 Timer。Timer 回调中重新加载 `session_context`（因为可能在回调触发时上下文已被修改）。
4. **幂等保证**：Phase 3 的去重基于 `mailid` 集合。该集合需持久化（文件或轻量数据库），避免 Agent 重启后重复处理邮件。
5. **错误处理**：每个阶段的 API 调用和文件 I/O 应包裹 try-catch，异常时记录日志并进入 Phase 6 闭环，不阻塞状态机。

### 伪代码：状态机主循环

```text
function run_session(cron_slot, config):
    session = create_session_context(cron_slot, config)
    session.phase = 1  // 采集

    while session.phase <= 6:
        if elapsed_since(session.session_started_at) > 30_minutes:
            session.phase = 6  // 超时强制闭环

        switch session.phase:
            case 1: phase_gather(session)
            case 2: phase_reach(session)
            case 3: phase_receive(session)
            case 4: phase_interpret(session)
            case 5: phase_act(session)
            case 6: phase_close(session); break
```

### 伪代码：Phase 3 的等待机制

Phase 3（接收）是唯一需要「等待」的阶段。实现上有两种模式：

**模式 A（同步轮询）**：Phase 3 内部使用轮询循环，每隔 30 秒检查一次收件箱，直到收到回复或超时。

```text
function phase_receive(session):
    start_polling()  // 切换到密集轮询间隔
    while session.phase == 3:
        new_mails = poll_inbox(since=session.last_sent_timestamp)
        for each mail in new_mails:
            if is_valid_reply(mail, session):
                session.user_message = extract_content(mail)
                cancel_dynamic_timers(session)
                if is_special_command(mail.content):
                    execute_command(mail.content)
                    session.phase = 6
                else:
                    session.phase = 4
                return
        sleep(poll_interval)  // 30s
        if elapsed > 30_minutes:
            session.phase = 6
            return
```

**模式 B（事件驱动）**：Phase 3 注册一个邮件到达的回调/事件处理器，由外部轮询服务驱动。Phase 3 进入后注册监听器，然后挂起等待通知。此模式更解耦但实现复杂度更高。

推荐使用模式 A（同步轮询），逻辑简单直观，适合单个 Agent 实例的场景。

### Claude Code 环境下的实现建议

如果 Life Coach Agent 运行在 Claude Code 环境中：

- 状态机的主循环由 Claude Code 的会话上下文管理（即每次被唤醒时从上次的 phase 继续）。
- Cron 触发由 `/loop` 命令或 settings.json hooks 配置（见 `engine/cron-system.md` 的 Claude Code 配置章节）。
- 动态 Timer（+5min/+10min）使用 Claude Code 的 `CronCreate` 工具注册一次性定时任务。
- 邮件轮询使用 MCP 工具 `poll_wecom_emails`。
- 邮件发送使用 MCP 工具 `send_wecom_email`。

## 文件完整性验证

运行以下命令确认本文件覆盖了所有核心阶段和边界情况：

### 状态机阶段检查

```bash
grep -c "Phase\|采集\|触达\|接收\|理解\|行动\|闭环" engine/session-flow.md
```

预期：至少匹配 6。

### 回复轮询检查

```bash
grep -c "5min\|10min\|轮询" engine/session-flow.md
```

预期：至少匹配 2。

### 边界情况覆盖检查

```bash
grep -c "无回复\|发送失败\|节假日\|边界情况" engine/session-flow.md
```

预期：至少匹配 2。

### 分类体系覆盖检查

```bash
grep -c "status_update\|emotional_signal\|modification_request\|deep_question\|create_intent\|rest_day_declaration\|task_commitment" engine/session-flow.md
```

预期：至少匹配 7（七种分类各自出现）。

## References

- `engine/cron-system.md`：Cron 调度规则，session-flow 的上游触发源和动态 Timer 规则。
- `engine/email-protocol.md`：邮件收发协议，session-flow 的消息通道和模板变量来源。
- `system/algorithm.md`：Agent 执行循环和推理框架，session-flow 状态机是其实例化。
- `system/memory-system.md`：记忆读写规则和维护周期，session-flow 在 Phase 1 和 Phase 6 中读写。
- `coach/coaching-process.md`：教练流程参考，Phase 5 emotional_signal 的情绪承接方法来源。
