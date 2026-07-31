# Cron 调度系统

本文件定义 Life Coach Agent 的时间驱动调度规则。所有 Cron 规则基于**工作日感知**：在法定工作日执行完整日间调度，在休息日/节假日执行轻量晚间复盘。

## 核心原则

1. **工作日全天陪伴**：日间 09:30-17:30 六个时间点提供主动引导，晚间 22:30 执行日终复盘。
2. **发送后异步跟进**：每次 Agent 主动发送消息后，在 +5min 和 +10min 两个时间点检查用户是否回复，实现「发完后等一等再提醒」的轻量跟进。
3. **休息日节奏可选**：周末和节假日默认使用简化时间骨架（4 个节点），Agent 基于用户的长期目标和已有承诺主动提供轻量提醒，用户可协商调整。当用户提前声明某天纯休息（如旅游、家庭日）时，系统切换为纯休息模式——仅保留日程事件提醒和体验记录，不做任务安排。
4. **可参数化**：所有时间点、启用/禁用、风格强度均暴露为用户可调参数。

## 时间点定义表

共 11 个时间点，分为五类：工作日日间节点（6 个，09:30/10:30/14:30/15:30/16:30/17:30）、休息日日间节点（3 个，10:00/14:30/17:30）、晚间节点（1 个，22:30）、发送后动态节点（2 个，+5min/+10min）。

每个 Cron 节点的工作分为两部分：**先回顾上一个执行块，再开启下一个执行块**。Agent 不盲目向前看，而是用上一个块的**实际执行数据**来校准判断。

| # | 时间点 | 触发方式 | 运行条件 | 回顾（Review） | 开启（Look Ahead） | 备注 |
|---|--------|---------|---------|-------------|----------------|------|
| 1 | **09:30** | Fixed Cron | 仅工作日 | 回顾昨日 `memory/daily/{yesterday}.md` 的复盘结论，若有未完成事项标记为「今天继续」；检查 `planning/commitments.md` 中今日到期及近日目标日期的承诺 | 早安问候 + 今日计划引导。读天气/日历/待办/commitments，询问今日关注点，列出今日到期承诺 | 一天的第一个节点，无同日前置块；此时提醒即将到期的承诺最自然 |
| 2 | **10:30** | Fixed Cron | 仅工作日 | 回顾 09:30 计划的实际执行情况：距离今日计划过去了 1 小时，用户在 09:30 承诺关注的事推进了多少？ | 上午状态检查：进度如何？精力状态？需要调整优先级吗？若有 commitments 中已过期的前置检查项，轻量提醒 | 如果第一个番茄钟就偏离了，不要等一天结束才发现 |
| 3 | **14:30** | Fixed Cron | 仅工作日 | 回顾上午整体执行：上午计划 vs 实际完成率。上午是否推进了 09:30 所关注的核心事项？是否有精力线索？ | 午后启动引导：下午最重要的 1-2 件事；检查本周末到期承诺的前置条件 | 用上午实际数据校准下午安排——不要盲目把上午没做的事原样压到下午 |
| 4 | **15:30** | Fixed Cron | 仅工作日 | 回顾 14:30-15:30 这一小时的推进：14:30 确定的下午 1-2 件事推进了吗？这个番茄钟的结果是什么？ | 下午中段状态检查：进度和精力检查，是否需要切换任务类型 | 不要让用户在同一个任务上卡了整个下午才被发现 |
| 5 | **16:30** | Fixed Cron | 仅工作日 | 回顾 15:30-16:30 执行结果 + 全天累计完成率：到今天下午此时，累计完成了今日计划的多少？和预期节奏一致吗？ | 收尾预备：距离日终 1 小时——今天还有什么必须在结束前推进的？检查 commitments 中是否有今天必须处理但用户未提到的事项 | 一天最后一个完整的执行窗口，需要清楚知道还有多少未了结 |
| 6 | **17:30** | Fixed Cron | 工作日 + 休息日 | 若为工作日：回顾全天执行，逐行回顾今日计划每项的完成状态、精力曲线和情绪模式。若为休息日：回顾今日承诺执行情况（哪些做了/哪些跳过/被取消） | 若为工作日：未完成项处置（延期/取消/明天）；明天最重要的 1 件事。若为休息日：未完成项处置；若次日为工作日则轻量衔接 | 休息日 17:30 不追进度，只做收尾和次日衔接 |
| 7 | **22:30** | Fixed Cron | 每日 | 回顾今日执行与情绪全貌；若为纯休息模式则回顾当日体验亮点 | 次日预览：若次日为工作日则提示查看日程、待办和 commitments；若为休息日则轻量提示下一周的承诺概览 | 三种模式的晚间复盘内容不同（见晚间节点行为章节） |
| R1 | **10:00** | Fixed Cron | 仅休息日/节假日 | 读取昨日 daily 和 commitments.md，筛选今日到期或标记「每日」的 step，以及未来 7 天内到期的里程碑 | 休息日晨间启动：主动列出今日相关承诺和长期目标拆解；若命中 rest_day 声明则切换为纯休息模式 | 比工作日晚 30 分钟，用户可能睡懒觉；不泛泛问「想干嘛」，而是基于已有数据主动提醒 |
| R2 | **14:30** | Fixed Cron | 仅休息日/节假日 | 检查上午是否有用户回复或执行记录；回顾 10:00 列出的事项 | 若用户已推进 → 肯定 + 问下午计划；若未回复 → 轻量提醒不催促；若已说不做 → 记录为纯休息，不再追问任务 | 休息日默认模式和纯休息模式的 14:30 内容不同 |
| R3 | **17:30** | Fixed Cron | 仅休息日/节假日 | （同 #6 休息日部分） | （同 #6 休息日部分） | 与工作日晚间收尾共享 17:30 槽位，内容根据模式切换 |
| 10 | **发送后+5min** | Dynamic Timer | 动态（发送后） | 检查用户是否回复了刚才的发送 | 未回复 → 温和提醒；已回复 → 取消 Timer | 不回顾上一个执行块——动态 Timer 是消息层面的跟进 |
| 11 | **发送后+10min** | Dynamic Timer | 动态（发送后） | 再次检查用户是否回复 | 归档为「等待用户回复」 | 同上 |

### 回顾规则（通用）

每个日间节点和日终节点的回顾遵循以下规则：

1. **回顾基于文件而非记忆**：读取 `memory/daily/{today}.md`（当日已有记录）、`planning/daily-plan.md`（当日计划）和 `planning/commitments.md`（跨天承诺追踪），而不是凭 Agent 的记忆。Agent 可能记错或会话跨了不同上下文（Cron 是独立触发的）。
2. **回顾必须具体**：不是泛泛问「做得怎么样？」，而是具体到上一个块的事实——「你在 09:30 说今天最关注 ___，过去 1 小时这个部分有进展吗？」
3. **偏离 = 数据，不 = 失败**：用上一步的实际完成率、实际推进量作为数据输入到 Algorithm 的 GAP 步骤，不做道德评判。
4. **非工作日的回顾风格不同**：
   - **休息日默认模式**：追踪承诺和长期目标拆解的完成情况，但语气轻松，不强制——用户说不做就调整，不做道德评判。回顾基于 commitments.md 中的 goal/step 数据，而非 daily-plan.md。
   - **纯休息模式**：不追踪任务，只做感受层面的轻量回顾和日程事件提醒。

### 时间点设计原理

- **09:30**：大多数用户已到达工作/学习环境，适合做全天规划。
- **10:30**：上午已进行约 1 小时，首次检查进度和精力，帮助用户修正认知偏差（例如「我以为 1 小时能做完」→「实际只完成了 30%」）。
- **14:30**：午休通常结束于 14:00 前后，给 30 分钟缓冲后做午后启动，符合精力曲线中下午的二次高峰定位。
- **15:30 / 16:30**：下午精力自然下降的两个观察点，分别关注「是否需要切换任务」和「收尾前的最后推进」。
- **17:30**：大多数工作/学习场景的结束时间，适合日终收尾。
- **22:30**：睡前约 1 小时，适合轻量复盘而不影响入睡。
- **+5min / +10min**：避免发送消息后立即催促。+5min 温和检查，+10min 归档，形成「发送 -> 等待 -> 温和提醒 -> 归档」的闭环。
- **休息日 10:00**：比工作日晚 30 分钟，给用户睡懒觉的缓冲。此时用户有足够时间自然醒来并思考今天的节奏。
- **休息日 14:30**：与工作日共享时间槽，但内容切换为轻量跟进——不追进度，只是确认用户是否需要推进承诺。
- **休息日缺少 10:30/15:30/16:30**：休息日不需要每小时番茄钟检查，自由节奏由用户主导。

## 工作日判断

### 主 API：timor.tech

调用 `https://timor.tech/api/holiday/info/{date}`，其中 `{date}` 格式为 `YYYY-MM-DD`（例如 `2026-07-29`）。

**API 返回结构：**

```json
{
  "type": {
    "type": 0,
    "name": "周三",
    "week": 3
  },
  "holiday": null
}
```

**判断逻辑：**

| `type.type` | `holiday` | 含义 | Agent 行为 |
|-------------|-----------|------|-----------|
| 0 | `null` | 普通工作日（周一至周五） | 全日间 5 个节点 + 晚间复盘 均执行 |
| 1 | `null` | 普通周末（周六、周日） | 日间跳过，仅晚间 22:30 轻量复盘 |
| 2 | 非 `null`（对象） | 法定节假日 | 日间跳过，仅晚间 22:30 轻量复盘 |
| 0 | 非 `null`（对象） | 法定调休补班日 | 按工作日处理，全日间节点均执行 |

**伪代码判断逻辑：**

```text
function isWorkday(date):
    response = GET https://timor.tech/api/holiday/info/{date}
    type = response.type.type
    holiday = response.holiday

    if type == 0 and holiday == null:
        return true   // 普通工作日
    if type == 0 and holiday != null:
        return true   // 调休补班日，按工作日处理
    if type == 2 and holiday != null:
        return false  // 法定节假日
    if type == 1 and holiday == null:
        return false  // 普通周末

    return false  // 兜底：未知类型按非工作日处理
```

**封装后的判断函数（供 session-flow 和所有 engine 层模块使用）：**

```text
function getDayType(date):
    response = GET https://timor.tech/api/holiday/info/{date}
    type = response.type.type
    holiday = response.holiday

    if type == 0 and holiday == null:
        return "workday"        // 普通工作日
    if type == 0 and holiday != null:
        return "workday"        // 调休补班日
    if type == 2 and holiday != null:
        return "holiday"        // 法定节假日，holiday.name 为节日名
    if type == 1 and holiday == null:
        return "weekend"        // 普通周末

    return "unknown"            // API 异常时降级为 unknown
```

### 备份 API：mxnzp.com

当 `timor.tech` 不可用时（网络超时、返回格式异常、状态码非 200），降级使用 `mxnzp.com`。

**调用方式：**

```text
GET https://www.mxnzp.com/api/holiday/single/{date}?app_id={APP_ID}&app_secret={APP_SECRET}
```

**配置要求：**

- `app_id` 和 `app_secret` 需要用户在 `integrations/tools.md` 中配置，或通过环境变量 `MXNZP_APP_ID` 和 `MXNZP_APP_SECRET` 注入。
- 未配置时，mxnzp.com 视为不可用，降级到本地日历推算。

**判断逻辑（mxnzp.com 返回结构不同，需单独适配）：**

```text
function isWorkdayMxnzp(date):
    response = GET https://www.mxnzp.com/api/holiday/single/{date}
               ?app_id={app_id}&app_secret={app_secret}

    if response.code != 0:
        return fallback_to_local(date)  // API 失败，降级本地推算

    holiday = response.data.holiday
    if holiday != null:
        return false  // 节假日

    // mxnzp 不返回 type.type，需结合星期判断
    weekday = response.data.weekday
    if weekday == "星期六" or weekday == "星期日":
        return false  // 周末

    return true  // 非节假日非周末 -> 工作日
```

### 本地日历推算（最终降级）

当两个 API 均不可用时，使用系统本地日历推算。

```text
function isWorkdayLocal(date):
    weekday = date.dayOfWeek()  // 1=Mon ... 7=Sun
    return weekday >= 1 and weekday <= 5  // 周一至周五视为工作日
```

注意：本地推算**无法感知法定节假日和调休**，误差较大。仅作为 API 完全不可用时的最后兜底。

### API 调用策略总结

```text
function getDayType(date):
    // 第 1 层：timor.tech（首选，无需 API Key）
    try:
        response = GET https://timor.tech/api/holiday/info/{date}
                timeout=5s
        if response.status == 200 and valid_json(response.body):
            return parse_timor(response.body)
    catch:
        pass  // 超时或网络错误，进入第 2 层

    // 第 2 层：mxnzp.com（备份，需 app_id/app_secret）
    if MXNZP_APP_ID and MXNZP_APP_SECRET:
        try:
            response = GET https://www.mxnzp.com/api/holiday/single/{date}
                    params { app_id, app_secret }, timeout=5s
            if response.status == 200 and valid_json(response.body):
                return parse_mxnzp(response.body)
        catch:
            pass

    // 第 3 层：本地推算（最终降级）
    return fallback_local(date)
```

## 假期模式行为

当 `getDayType()` 返回 `"weekend"` 或 `"holiday"` 时，进入假期模式。

### 休息日时间骨架

休假不意味着没有节奏，只是节奏和核心不同。休息日使用一个**简化版时间骨架**（3 个节点，而非工作日的 7 个）：

| # | 时间点 | 触发方式 | 运行条件 | 触发动作 | 备注 |
|---|--------|---------|---------|---------|------|
| W1 | **10:00** | Fixed Cron | 仅休息日/节假日 | 休息日晨间问候 | 不包含任务规划——包含：「早安。今天不用赶，如果有什么想做的放松小事，可以告诉我。」 |
| W2 | **17:00** | Fixed Cron | 仅休息日/节假日 | 休息日下午回顾 | 轻量回顾：「今天过得怎么样？有没有一个让你觉得不错的时刻？」 |
| W3 | **22:30** | Fixed Cron | 每日 | 晚间轻量复盘 / 次日预览 | 如次日为工作日：提示「明天是工作日了，需要我帮你看看明天的日程和待办吗？」；如次日为休息日：极简问候「晚安，今天好吗？」 |

**休息日和工作日时间骨架的对比：**

| 维度 | 工作日 | 休息日 |
|------|--------|--------|
| 时间节点数 | 7 个（09:30-22:30，每小时番茄钟节奏） | 3 个（10:00 / 17:00 / 22:30） |
| 晨间节点 | 09:30 早安 + 天气 + 日历 + 计划引导 | 10:00 轻量问候（睡个懒觉，晚一点再问） |
| 日间节点 | 每小时一个检查点（10:30/14:30/15:30/16:30） | 不设置——休息日的自由节奏由用户主导 |
| 下午节点 | 多个进度检查 | 17:00 一个轻量回顾，不再追问 |
| 晚间节点 | 22:30 复盘执行与觉察 | 22:30 复盘当日感受，含次日预览 |
| 消息风格 | `cron.style`（warm/brief/challenge） | 始终 `warm`，不可切换 |
| 任务驱动 | 是——每个节点对照今日计划 | 否——当天不追踪任务进度 |

### 休息日的规划逻辑

**前一天晚间（22:30 复盘时）**：如果次日是休息日，Agent 在当日复盘末尾追加：

```text
明天是休息日。有什么想做的放松小事吗？（也可以不安排，随意过一天。）
```

Agent 可以在 `memory/daily/{tomorrow}.md` 中创建一个极简条目：

```markdown
# {date} 休息日

## 想做的放松小事（可选）
- （用户表达的内容）

## 实际感受（晚间填写）
```

**休息日当天**：
- 10:00 节点读取上述文件，如果用户有写想做的事，可以轻量引导：「昨天你提到想 ___，今天有这个心情吗？」
- 17:00 节点轻量询问今天感受。
- 22:30 节点做轻量复盘，并检查次日是否为工作日，若是则做工作日衔接。

**休息日结束后第一个工作日 09:30**：已有机�的假期回归版问候，但需额外做以下连接：

1. 读取休息日的 `memory/daily/{rest-day}.md`，了解休息日状态。
2. 在早安问候中轻量连接：「假期过得怎么样？今天回到工作节奏，先别急着把所有事都捡起来——今天最重要的 1 件事是什么？」
3. 读取原有的 `planning/daily-plan.md` 中可能已被假日打破的安排，帮助用户重新安排今日计划。

### 休息日参数化配置

| 参数 | 默认值 | 说明 | 示例 |
|------|--------|------|------|
| `cron.rest_day_morning_time` | `10:00` | 休息日晨间问候时间 | `09:00` / `11:00` |
| `cron.rest_day_afternoon_time` | `17:00` | 休息日下午回顾时间 | `16:00` / `18:00` |
| `cron.rest_day_morning_enabled` | `true` | 启用休息日晨间问候（设为 `false` 则休息日完全静默，仅晚间复盘） | `true` / `false` |
| `cron.rest_day_afternoon_enabled` | `true` | 启用休息日下午回顾 | `true` / `false` |

### 日间节点行为（原有静默规则整合）

所有工作日日间节点（09:30 / 10:30 / 14:30 / 15:30 / 16:30 / 17:30）在休息日**静默跳过**，不发送任何消息。休息日使用上方的休息日时间骨架（10:00 / 17:00 / 22:30）。

### 晚间节点行为

22:30 节点照常执行，但消息风格切换为轻量复盘模式：

**工作日晚间复盘（22:30）：**

```text
今天的三个回顾：
1. 完成了什么？（哪怕一小步）
2. 有什么偏离计划的地方？（把它当数据，不当失败）
3. 明天最重要的 1 件事是？
```

**休息日晚间轻量回顾（22:30）：**

```text
今天休息得怎么样？
- 有没有一个让你放松或开心的瞬间？
- 明天有什么想做的小事？

（不需要答案，只是陪你回顾一下今天。）
```

**节假日特别版（22:30，当 holiday.name 有值时）：**

```text
今天是{holiday.name}，节日快乐。
今天有没有一个让你觉得「今天真不错」的瞬间？
明天有什么期待的小事？
```

### 假期连续提醒抑制

当连续多天处于假期模式时（例如春节、国庆），每日晚间复盘不重复发送相同内容。Agent 在假期第 1 天发送复盘引导后，后续假期日仅在 22:30 发送一条极简问候：

```text
晚安。今天过得好吗？
```

连续假期结束后第一个工作日（即 `getDayType()` 从非工作日翻转为工作日），09:30 的早安问候切换为假期回归版：

```text
早上好！假期结束了，今天回到工作节奏。
先别急着把所有事都捡起来——今天最重要的 1 件事是什么？
需要我帮你看看这段时间的待办和日程吗？
```

## 用户可参数化配置

以下配置项暴露为用户可调参数，建议存储在 `profile/user.md` 的 `## Cron配置` 章节，或通过 `integrations/tools.md` 中的环境变量注入。

| 参数 | 默认值 | 说明 | 示例 |
|------|--------|------|------|
| `cron.enabled` | `true` | 全局 Cron 开关。`false` 时所有定时节点静默，按需手动触发。 | `true` / `false` |
| `cron.timezone` | `Asia/Shanghai` | 时区，影响所有 Fixed Cron 的触发时间。 | `Asia/Shanghai` / `America/New_York` |
| `cron.morning_time` | `09:30` | 早安问候时间（HH:MM）。 | `08:00` / `09:00` / `10:00` |
| `cron.mid_morning_time` | `10:30` | 上午状态检查时间。设为 `""` 可禁用此节点。 | `10:00` / `""` |
| `cron.after_lunch_time` | `14:30` | 午后启动引导时间。 | `14:00` / `15:00` |
| `cron.mid_afternoon_time` | `15:30` | 下午中段检查时间。设为 `""` 可禁用。 | `15:00` / `""` |
| `cron.pre_wrap_time` | `16:30` | 收尾预备时间。设为 `""` 可禁用。 | `16:00` / `""` |
| `cron.evening_wrap_time` | `17:30` | 日终收尾时间。 | `18:00` / `17:00` |
| `cron.night_review_time` | `22:30` | 晚间复盘时间。 | `22:00` / `23:00` |
| `cron.followup_5min_enabled` | `true` | 启用发送后+5min 跟进行为。 | `true` / `false` |
| `cron.followup_10min_enabled` | `true` | 启用发送后+10min 归档行为。 | `true` / `false` |
| `cron.style` | `warm` | 消息风格。「warm」为温暖陪伴风格，「brief」为简洁要点风格，「challenge」为适度挑战风格。 | `warm` / `brief` / `challenge` |
| `cron.holiday_night_review` | `true` | 休息日/节假日是否发送晚间复盘。 | `true` / `false` |
| `cron.holiday_consecutive_suppress` | `true` | 连续假日是否抑制重复复盘。`true` 时，假期第 2 天起极简问候。 | `true` / `false` |

### 配置文件示例（`profile/user.md` 中的 Cron 配置段）

```markdown
## Cron配置

- `cron.enabled`: true
- `cron.timezone`: Asia/Shanghai
- `cron.morning_time`: 09:30
- `cron.mid_morning_time`: 10:30
- `cron.after_lunch_time`: 14:30
- `cron.mid_afternoon_time`: 15:30
- `cron.pre_wrap_time`: 16:30
- `cron.evening_wrap_time`: 17:30
- `cron.night_review_time`: 22:30
- `cron.followup_5min_enabled`: true
- `cron.followup_10min_enabled`: true
- `cron.style`: warm
- `cron.holiday_night_review`: true
- `cron.holiday_consecutive_suppress`: true
```

## AI 工具配置 Cron 的指导说明

本文件定义的 Cron 规则是**声明式的时间调度规范**，具体实现取决于宿主 AI 工具的 Cron/定时任务能力。以下是主流工具的配置参考。

### 通用原则

1. 所有 Fixed Cron 的时间以 `cron.timezone`（默认 `Asia/Shanghai`）为基准。
2. 每个时间点在触发时应先调用 `getDayType(today)`，仅当返回 `"workday"` 时才执行日间节点逻辑。
3. 发送后+5min 和+10min 是动态 Timer，不是 Fixed Cron：它们在 Agent 每次主动发送消息后由 session-flow 动态设置，消息上下文结束后自动清除。
4. API 降级链路（timor.tech -> mxnzp.com -> 本地推算）应有日志记录，便于排查工作日判断异常。

### Claude Code 定时任务（通过 /loop 命令或 hooks 配置）

Claude Code 支持通过 `/loop` 命令或 `settings.json` 中的 hooks 配置定时触发。

**通过 /loop 命令注册（一次性，适合快速验证）：**

```text
/loop 30 9 * * 1-5 早安问候——检查今天是否是工作日，如果是则发送今日计划引导。
/loop 30 10 * * 1-5 上午状态检查。
/loop 30 14 * * 1-5 午后启动引导。
/loop 30 15 * * 1-5 下午中段检查。
/loop 30 16 * * 1-5 收尾预备。
/loop 30 17 * * 1-5 日终收尾。
/loop 30 22 * * * 晚间复盘（含休息日轻量版）。
```

注意：`/loop` 命令在 Claude Code 会话中注册的定时任务依赖 `cron.enabled` 为 `true` 且会话保持活动。适用于 AI 持续在线（例如 webhook 或长期会话）的场景。

**通过 settings.json 配置 hooks（推荐，可持久化）：**

```json
{
  "hooks": {
    "CronTrigger": [
      {
        "schedule": "30 9 * * 1-5",
        "command": "echo 'morning_greeting'",
        "description": "09:30 早安问候"
      },
      {
        "schedule": "30 10 * * 1-5",
        "command": "echo 'mid_morning_check'",
        "description": "10:30 上午状态检查"
      },
      {
        "schedule": "30 14 * * 1-5",
        "command": "echo 'after_lunch_start'",
        "description": "14:30 午后启动"
      },
      {
        "schedule": "30 15 * * 1-5",
        "command": "echo 'mid_afternoon_check'",
        "description": "15:30 下午中段检查"
      },
      {
        "schedule": "30 16 * * 1-5",
        "command": "echo 'pre_wrap_check'",
        "description": "16:30 收尾预备"
      },
      {
        "schedule": "30 17 * * 1-5",
        "command": "echo 'evening_wrap'",
        "description": "17:30 日终收尾"
      },
      {
        "schedule": "30 22 * * *",
        "command": "echo 'night_review'",
        "description": "22:30 晚间复盘"
      }
    ]
  }
}
```

### 飞书 / Lark 机器人定时消息

如果 Life Coach Agent 通过飞书机器人发送消息，可使用飞书开放平台的「定时消息」或外部 Cron 服务触发机器人 webhook。

**Cron 表达式参考（标准 5 字段）：**

| 时间点 | Cron 表达式 | 说明 |
|--------|-----------|------|
| 09:30 | `30 9 * * 1-5` | 工作日（周一至周五）09:30 |
| 10:30 | `30 10 * * 1-5` | 工作日 10:30 |
| 14:30 | `30 14 * * 1-5` | 工作日 14:30 |
| 15:30 | `30 15 * * 1-5` | 工作日 15:30 |
| 16:30 | `30 16 * * 1-5` | 工作日 16:30 |
| 17:30 | `30 17 * * 1-5` | 工作日 17:30 |
| 22:30 | `30 22 * * *` | 每天 22:30 |

注意：Cron 表达式中的星期字段 `1-5` 只是粗筛（周一至周五），**每个时间点触发后仍需调用 `getDayType()` 检查是否为法定节假日或调休日**，避免在法定节假日发送日间消息，或漏掉调休补班日。

### 自建 Cron 服务（Node.js / Python / 云函数）

如果自建 Cron 调度服务，每个时间点触发的处理函数应遵循统一模板：

```python
# 伪代码：每个 Cron 触发的处理函数结构
def on_cron_trigger(time_slot: str, config: dict):
    if not config.get("cron.enabled", True):
        return  # 全局开关关闭

    today = get_today_string()  # "YYYY-MM-DD"
    day_type = get_day_type(today)  # 调用 timor.tech 降级链路

    # 日间节点：仅工作日执行
    if time_slot in ["morning", "mid_morning", "after_lunch", "mid_afternoon", "pre_wrap", "evening_wrap"]:
        if day_type == "workday":
            send_message(time_slot, config)
        else:
            log(f"跳过 {time_slot}: 非工作日 ({day_type})")
        return

    # 晚间节点：每日执行，但消息内容根据 day_type 切换
    if time_slot == "night_review":
        if day_type in ["weekend", "holiday"]:
            if not config.get("cron.holiday_night_review", True):
                return
            send_holiday_night_message(day_type, config)
        else:
            send_workday_night_message(config)
        return
```

### 发送后动态 Timer 实现说明

发送后+5min 和+10min 不是 Fixed Cron，而是动态的一次性 Timer。实现要点：

1. Agent 每次通过消息通道主动发送消息后，在 session-flow 中记录 `last_sent_message_id` 和 `last_sent_timestamp`。
2. 同时设置一个 5 分钟后的一次性 Timer（`set_timeout(5min, check_reply)`）。
3. 在 `check_reply` 中：检查自 `last_sent_message_id` 以来用户是否已回复。如果没有新用户消息，发送温和提醒。
4. 同时设置一个 10 分钟后的一次性 Timer（`set_timeout(10min, archive_waiting)`）。在 `archive_waiting` 中：如果用户仍未回复，归档为「等待用户回复」状态。
5. 如果用户在 +5min 或 +10min Timer 触发前回复了，两个 Timer 自动取消。

```python
# 动态 Timer 伪代码
def after_agent_send_message(message_id: str):
    session.last_sent_message_id = message_id
    session.last_sent_timestamp = now()

    # 如果启用 +5min 跟进
    if config.get("cron.followup_5min_enabled", True):
        schedule_once(delay=5min, callback=lambda: check_user_reply_5min(message_id))

    # 如果启用 +10min 归档
    if config.get("cron.followup_10min_enabled", True):
        schedule_once(delay=10min, callback=lambda: archive_if_no_reply_10min(message_id))

def check_user_reply_5min(sent_message_id: str):
    if no_new_user_message_since(sent_message_id):
        send_gentle_reminder()  // "刚才的消息看到了吗？不着急，想到了随时回。"

def archive_if_no_reply_10min(sent_message_id: str):
    if no_new_user_message_since(sent_message_id):
        archive_session("waiting_for_reply")  // 归档，不继续发送消息
```

## Cron 规则验证

### 完整性检查

运行以下命令确认本文件的调度规则覆盖所有预期时间点：

```bash
grep -c "09:30\|10:30\|14:30\|15:30\|16:30\|17:30\|22:30" engine/cron-system.md
```

预期：至少匹配 7 个时间点。

### API 参考检查

```bash
grep -c "timor.tech\|mxnzp" engine/cron-system.md
```

预期：至少匹配 2。

### 工作日语义检查

```bash
grep -c "工作日\|休息日\|假期\|节假日" engine/cron-system.md
```

预期：至少匹配 4。

## 与其他 engine 模块的关系

- **session-flow.md**：读取本文件的调度规则，在每个 Cron 触发时调用 `getDayType()` 判断是否执行；发送消息后设置 +5min/+10min 动态 Timer；Cron 触发时先检查收件箱（Phase 1 Step 4.5）确保不遗漏用户已发送但未处理的回复。
- **email-protocol.md**：当 Cron 触发的消息需要邮件渠道发送时（例如非 IM 场景），通过 email-protocol 的模板规则组装邮件内容；收件箱检查使用 email-protocol 的 `list_mail` / `get_mail_content` API + 去重集合。

## 维护说明

- 时间点调整：修改 `## 用户可参数化配置` 中的对应参数默认值，或让用户在 `profile/user.md` 中覆盖。
- 新增时间点：在时间点定义表中添加新行，更新 Cron 表达式参考表，补充对应的触发内容逻辑。
- API 地址变更：更新 `## 工作日判断` 中的 API URL 和判断逻辑；备份 API 如有新增也在此补充。
- 风格自定义：`cron.style` 参数的具体消息模板在 `coach/messages/` 目录下单独维护，本文件只声明参数不绑定模板内容。

## References

- `engine/session-flow.md`：会话流控制，读取 Cron 规则驱动对话节奏。
- `engine/email-protocol.md`：邮件渠道的消息组装规则。
- `integrations/tools.md`：外部 API Key（如 mxnzp.com 的 app_id/app_secret）的配置位置。
- `profile/user.md`：用户覆写的 Cron 配置参数存放处。
