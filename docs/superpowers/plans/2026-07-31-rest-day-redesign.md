# 休息日调度重设计 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将休息日从「不做任务、不打扰」重设计为「默认可推进长期目标，用户主动声明则切纯休息模式」。

**Architecture:** 修改 6 个文件：重写 cron-system.md 的休息日行为（三模式、4 节点、rest_day 覆盖逻辑），扩展 commitments.md 数据模型（类型字段、父承诺字段、goal/step 层级拆解、归档规则），新建 commitments-archive.md，更新 session-flow.md（Phase 4/5/1 适配新类型），更新 algorithm.md（CHECK/GAP 三模式差异），更新 AGENTS.md 路由表。不改动 coach skills、daily-plan 模板、email-protocol 和 integrations。

**Tech Stack:** Markdown 文档工程（协议/模板定义），无运行时代码。

## Global Constraints

- 所有 Cron 节点触发前必须先通过 `getDayType()` × `commitments.md` rest_day 检测确定最终模式
- 休息日默认模式：4 个 Cron 节点（10:00/14:30/17:30/22:30），基于长期目标 + 承诺主动提醒
- 纯休息模式：同样 4 个 Cron 节点但切除任务安排，仅保留日程事件提醒和体验记录
- `rest_day` 声明存入 commitments.md，新增 `类型` 字段区分 plan/rest_day/goal/step
- 新增 `父承诺` 字段建立 goal → step 层级关系
- 目标日期为「每日」的 step 每天都提醒，用户可每天说不做
- 归档规则：已完成/取消超过 14 天自动移入 commitments-archive.md；过期超过 30 天确认后归档
- 活跃行 ≤ 20 行

---

### Task 1: 重写 cron-system.md 休息日章节

**Files:**
- Modify: `engine/cron-system.md`（多处修改）

**Interfaces:**
- Consumes: 设计文档 `docs/superpowers/specs/2026-07-31-rest-day-redesign.md`
- Produces: 更新后的 Cron 调度规范，定义了三种模式（工作日/休息日默认/纯休息）、4 个休息日节点、`getDayType()` rest_day 覆盖逻辑、更新参数表

- [ ] **Step 1: 更新核心原则章节（第 3-4 行）**

将第 3 行的「休息日不打扰」原则替换为新的休息日处理原则。定位第 3 行：

```
3. **休息日不打扰**：周末和节假日白天静默，仅保留晚间 22:30 的轻量复盘提示（可回顾今天感受、明天准备）。
```

替换为：

```markdown
3. **休息日节奏可选**：周末和节假日默认使用简化时间骨架（4 个节点），Agent 基于用户的长期目标和已有承诺主动提供轻量提醒，用户可协商调整。当用户提前声明某天纯休息（如旅游、家庭日）时，系统切换为纯休息模式——仅保留日程事件提醒和体验记录，不做任务安排。
```

- [ ] **Step 2: 更新时间点定义表（第 18-28 行）**

将当前 9 个时间点的定义表替换为包含休息日节点的完整定义表。

定位第 16-28 行的整张表（`| # | 时间点 | ...` 到注释行），替换为：

```markdown
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
```

- [ ] **Step 3: 更新回顾规则章节（第 32-37 行）**

在第 4 条「休息日的回顾风格不同」后增加模式区分。

定位第 37 行：

```
4. **休息日的回顾风格不同**：休息日不追踪任务完成率，只做感受层面的轻量回顾（见假期模式行为一节）。
```

替换为：

```markdown
4. **非工作日的回顾风格不同**：
   - **休息日默认模式**：追踪承诺和长期目标拆解的完成情况，但语气轻松，不强制——用户说不做就调整，不做道德评判。回顾基于 commitments.md 中的 goal/step 数据，而非 daily-plan.md。
   - **纯休息模式**：不追踪任务，只做感受层面的轻量回顾和日程事件提醒。
```

- [ ] **Step 4: 更新时间点设计原理（第 40-47 行）**

在 10:30 原理后面增加休息日节点的设计原理。

定位第 47 行（`- **+5min / +10min**：...` 之后），追加：

```markdown
- **休息日 10:00**：比工作日晚 30 分钟，给用户睡懒觉的缓冲。此时用户有足够时间自然醒来并思考今天的节奏。
- **休息日 14:30**：与工作日共享时间槽，但内容切换为轻量跟进——不追进度，只是确认用户是否需要推进承诺。
- **休息日缺少 10:30/15:30/16:30**：休息日不需要每小时番茄钟检查，自由节奏由用户主导。
```

- [ ] **Step 5: 重写「假期模式行为」章节（第 193-261 行）**

这是最大的改动。将第 193 行到第 261 行的整个「假期模式行为」章节替换为新版本。

定位第 193 行 `## 假期模式行为` 到第 261 行 `### 晚间节点行为` 之前的内容（即一整节），替换为：

```markdown
## 非工作日模式

当 `getDayType()` 返回 `"weekend"` 或 `"holiday"` 时，进入非工作日模式。非工作日模式分为两个子模式：

- **休息日默认模式**：用户未声明当天纯休息，Agent 基于长期目标和已有承诺主动提供轻量提醒。
- **纯休息模式**：用户在 commitments.md 中声明了当天为 `rest_day`，Agent 仅保留日程事件提醒和体验记录。

### 模式判定逻辑

每个 Cron 节点触发时，先判定最终模式：

```text
1. cal_type = getDayType(today)   // 日历类型：workday / weekend / holiday / makeup / unknown
2. user_rest = 查询 commitments.md：
   - 目标日期 == today AND 类型 == "rest_day" AND 状态 == "pending"
3. 最终模式判定：

   if cal_type == "workday" and not user_rest:
       mode = "workday"            // 工作模式：7 节点全功能
   if cal_type == "workday" and user_rest:
       mode = "pure_rest"          // 工作日请假场景
   if cal_type in ("weekend", "holiday") and not user_rest:
       mode = "rest_day_default"   // 休息日默认：4 节点，轻量提醒可工作
   if cal_type in ("weekend", "holiday") and user_rest:
       mode = "pure_rest"          // 纯休息
```

### 三种模式对比

| 维度 | 工作模式 | 休息日默认模式 | 纯休息模式 |
|------|---------|-------------|----------|
| 触发条件 | 周一-周五（含调休补班） | 周末/节假日，无 rest_day 声明 | 用户声明了某天为 rest_day |
| Cron 节点 | 7 个（09:30/10:30/14:30/15:30/16:30/17:30/22:30） | 4 个（10:00/14:30/17:30/22:30） | 4 个（10:00/14:30/17:30/22:30） |
| 任务提醒 | 对照日计划追踪进度 | 基于长期目标 + 承诺主动提醒，可协商 | 不追任务 |
| 日程事件提醒 | 有 | 有 | 有 |
| 体验记录 | 有 | 有 | 有 |
| 消息风格 | warm/brief/challenge 可切换 | 轻松提醒，用户说不做就调整 | 纯陪伴，不推进任何工作 |

### 休息日时间骨架

休息日使用一个简化版时间骨架（4 个节点，而非工作日的 7 个）。去掉了每小时番茄钟检查（10:30/15:30/16:30），保留有意义的里程碑节点。

#### R1: 10:00 — 晨间启动

**运行条件**：仅休息日/节假日。若命中 rest_day 声明则切换为纯休息模式。

**回顾**：
- 读取昨日 `memory/daily/{yesterday}.md`
- 读取 `planning/commitments.md`，筛选：
  - 目标日期 == today AND 状态 == pending 的 plan/goal/step
  - 目标日期 == "每日" AND 状态 == pending 的 step
  - 目标日期在未来 7 天内的 step（预览性）
- 检查 rest_day 声明：目标日期 == today AND 类型 == "rest_day"

**开启**：

若为**休息日默认模式**：
```
早安问候（比工作日晚，用户可能睡懒觉）。
主动列出今日相关承诺和长期目标拆解：
  「早上好。今天有你之前提到的 ___（承诺），
    还有一个从 ___（长期目标）拆出来的 ___。
    有想推进的吗？（不想做也没关系，告诉我一声我就调整。）」

若用户回复「不想做」 → 记录跳过，调整后续提醒（今日后续节点降级为纯休息）
若用户回复「只做 ___」 → 聚焦该项，其余不追问
若用户未回复 → 14:30 节点轻量跟进
```

若为**纯休息模式**：
```
轻量问候：「早安。今天{rest_day 承诺的内容摘要，如"去南京"}，玩得开心！🎒」
+ 日程事件主动提醒：「对了，你 _:__ 的 ___ 别忘了～」
```

**备注**：不泛泛问「今天想干嘛」，而是基于 commitments.md 中已有的数据主动提醒。

#### R2: 14:30 — 午后跟进

**运行条件**：仅休息日/节假日。

**回顾**：
- 检查上午是否有用户回复或执行记录
- 回顾 10:00 列出的事项

**开启**：

若为**休息日默认模式**：
```
若用户已推进 → 肯定 + 问下午计划
  「上午把 ___ 搞定了，不错。下午还想做什么吗？」
若用户未回复 → 轻量提醒（不催促）
  「刚才提到的 ___ 还想做吗？不想做的话告诉我一声就好～」
若用户上午已说不做 → 记录调整为纯休息，不再追问任务
```

若为**纯休息模式**：
```
纯陪伴，不追问任务：
  「在 ___了\？有什么新发现吗？」
若有日程事件发生中 → 记录到 daily
```

#### R3: 17:30 — 日终收尾

**运行条件**：工作日 + 休息日共享。内容根据模式切换。

**回顾**：
- 若为工作日：回顾全天执行（同原 #6）
- 若为休息日默认：回顾今日承诺执行情况（哪些做了/跳过/取消）
- 若为纯休息：回顾当日体验亮点

**开启**：
- 未完成项处置：延期/取消/明天
- 若次日为工作日 → 轻量衔接：「明天是工作日了，今天的 ___ 要延到明天吗？」
- 若次日为休息日 → 极简：「今天有什么收获吗？哪怕是很小的。」
- 若长期目标拆解的 step 今日未推进 → 询问是否重新排期

#### 晚间 22:30

22:30 每日触发，三种模式内容不同（见晚间节点行为章节）。

### 休息日参数化配置

| 参数 | 默认值 | 说明 | 示例 |
|------|--------|------|------|
| `cron.rest_day_morning_time` | `10:00` | 休息日晨间启动时间 | `09:00` / `11:00` |
| `cron.rest_day_morning_enabled` | `true` | 启用休息日晨间启动（设为 `false` 则休息日跳过晨间，仅保留 14:30/17:30/22:30） | `true` / `false` |
| `cron.rest_day_default_mode` | `default` | 休息日未声明时的默认模式：`default`（可工作）或 `pure_rest`（纯休息，回到旧行为） | `default` / `pure_rest` |

### 日间节点行为（原有静默规则整合）

所有工作日专属日间节点（09:30 / 10:30 的工作日版 / 15:30 / 16:30）在休息日**静默跳过**，不发送任何消息。休息日使用上方的休息日时间骨架（10:00 / 14:30 / 17:30 / 22:30）。

### 晚间节点行为

22:30 节点照常执行，内容根据当日最终模式切换：

**工作日晚间复盘（22:30）：**

```text
今天的三个回顾：
1. 完成了什么？（哪怕一小步）
2. 有什么偏离计划的地方？（把它当数据，不当失败）
3. 明天最重要的 1 件事是？
```

**休息日默认模式晚间回顾（22:30）：**

```text
今天有什么收获吗？
- 推进了什么？（如果有）
- 有什么让你放松或开心的瞬间？
- 明天有什么想做的小事？（若次日为休息日）
- 明天是工作日了，需要看一下日程和待办吗？（若次日为工作日）

轻量提示：未来一周的承诺概览（「下周有个 ___，有没有需要提前准备的？」）
```

**纯休息模式晚间回顾（22:30）：**

```text
今天{rest_day 内容，如"在南京"}怎么样？
- 有没有一个让你觉得「今天真不错」的瞬间？
- 明天有什么期待的小事？

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

### 休息日前一晚的衔接

**前一天晚间（22:30 复盘时）**：如果次日是休息日且为休息日默认模式：

```text
明天是休息日。我看到你有 ___ 和 ___ 的计划/拆解步骤在明天。
想做哪个？（也可以都不做——告诉我就行。）
```

Agent 不创建休息日日计划文件。提醒直接来自 commitments.md 的 goal/step 数据。
```

- [ ] **Step 6: 更新 session-flow.md 中引用的休息日跳过逻辑**

此步骤在 Task 4 中执行，此处仅确认 cron-system.md 的修改已完成。

- [ ] **Step 7: 更新用户可参数化配置表（第 311-328 行）**

参数表保持不变（现有参数足以支持新模式，无需新增）。仅在 `cron.holiday_consecutive_suppress` 描述后确认无误。

- [ ] **Step 8: 更新 Cron 表达式参考表（第 428-440 行）**

在工作日日间节点的 Cron 表达式后面，追加休息日节点的 Cron 表达式：

```
| 时间点 | Cron 表达式 | 说明 |
|--------|-----------|------|
| 09:30 | `30 9 * * 1-5` | 工作日 09:30 |
| 10:30 | `30 10 * * 1-5` | 工作日 10:30 |
| 14:30 | `30 14 * * 1-5` | 工作日 14:30；**休息日 14:30 也在此触发，内容根据模式切换** |
| 15:30 | `30 15 * * 1-5` | 工作日 15:30 |
| 16:30 | `30 16 * * 1-5` | 工作日 16:30 |
| 17:30 | `30 17 * * 1-5` | 工作日 17:30；**休息日 17:30 也在此触发，内容根据模式切换** |
| 10:00 | `0 10 * * 6,0` | 休息日晨间启动（周六日 10:00） |
| 22:30 | `30 22 * * *` | 每天 22:30 |
```

- [ ] **Step 9: 更新自建 Cron 服务伪代码（第 446-473 行）**

将旧的 `on_cron_trigger` 伪代码替换为支持三模式的版本：

```python
# 伪代码：每个 Cron 触发的处理函数结构
def on_cron_trigger(time_slot: str, config: dict):
    if not config.get("cron.enabled", True):
        return

    today = get_today_string()
    cal_type = get_day_type(today)  # 日历类型
    user_rest = check_rest_day_declaration(today)  # 查询 commitments.md
    mode = determine_mode(cal_type, user_rest)

    # 工作日专属日间节点
    if time_slot in ["morning", "mid_morning", "after_lunch_workday", "mid_afternoon", "pre_wrap"]:
        if mode == "workday":
            send_message(time_slot, config, mode)
        else:
            log(f"跳过 {time_slot}: 非工作模式 ({mode})")
        return

    # 休息日日间节点
    if time_slot in ["rest_morning", "rest_afternoon"]:
        if cal_type in ["weekend", "holiday"]:
            send_message(time_slot, config, mode)
        else:
            log(f"跳过 {time_slot}: 非休息日 ({cal_type})")
        return

    # 共享节点（14:30/17:30/22:30）—— 内容和模式切换在 send_message 内部处理
    if time_slot in ["after_lunch_shared", "evening_wrap_shared", "night_review"]:
        if cal_type in ["weekend", "holiday"] and config.get("cron.holiday_night_review", True) == False:
            if time_slot == "night_review":
                return
        send_message(time_slot, config, mode)
        return

def determine_mode(cal_type, user_rest):
    if cal_type == "workday" and not user_rest:
        return "workday"
    if cal_type == "workday" and user_rest:
        return "pure_rest"
    if cal_type in ("weekend", "holiday") and not user_rest:
        return "rest_day_default"
    if cal_type in ("weekend", "holiday") and user_rest:
        return "pure_rest"
    return "workday"  # unknown 降级
```

- [ ] **Step 10: 更新维护说明（第 544-547 行）**

在第 544 行之前追加：

```markdown
- 休息日模式调整：修改 `## 非工作日模式` 中的模式判定逻辑和节点行为。
- 新增休息日节点：在时间点定义表中添加 R 编号行，更新 Cron 表达式参考表。
```

- [ ] **Step 11: 验证 cron-system.md 修改的完整性**

检查以下关键词是否覆盖：
```bash
grep -c "rest_day\|纯休息\|休息日默认\|determine_mode\|rest_day_default\|pure_rest" engine/cron-system.md
```

预期：至少匹配 8。

- [ ] **Step 12: Commit**

```bash
git add engine/cron-system.md
git commit -m "feat: redesign rest day scheduling with three modes and four cron slots"
```

---

### Task 2: 扩展 commitments.md 模板（类型字段、父承诺、goal/step、归档规则）

**Files:**
- Modify: `templates/planning/commitments.md`

**Interfaces:**
- Consumes: 设计文档中的 goal/step 拆解模型和归档规则
- Produces: 更新后的 commitments 模板，新增 `类型` 和 `父承诺` 字段，新增 goal/step 拆解说明，新增自动归档规则

- [ ] **Step 1: 更新待办承诺表头（第 22-25 行）**

将当前表头：

```markdown
| # | 承诺 | 目标日期/时间 | 来源对话 | 前置检查 | 提醒时间 | 状态 |
|---|------|-------------|---------|---------|---------|------|
| 1 |  | YYYY-MM-DD | 对话/邮件 | 需要在此日期前做什么？ | 提前提醒的 Cron 节点 | pending / done / cancelled |
```

替换为：

```markdown
| # | 类型 | 承诺 | 目标日期 | 父承诺 | 来源对话 | 前置检查 | 提醒时间 | 状态 |
|---|------|------|---------|--------|---------|---------|---------|------|
| 1 |  |  | YYYY-MM-DD | - | 对话/邮件 | 需要在此日期前做什么？ | 提前提醒的 Cron 节点 | pending / done / cancelled |
```

- [ ] **Step 2: 更新字段说明（第 26-33 行）**

将当前字段说明替换为：

```markdown
**字段说明：**

- **#**：唯一编号。goal 用整数，其 step 用 `{goal号}.{序号}`（如 `1.1`、`1.2`），父子关系可读。
- **类型**：`plan`（普通承诺）、`goal`（长期目标）、`step`（目标拆解步骤）、`rest_day`（全天休息声明）。
- **承诺**：用户說的具体承诺，尽量引用原话。
- **目标日期**：承诺执行的日期。特殊值：`每日` 表示该 step 每天都出现在提醒列表中；`YYYY-MM-DD` 为具体日期。
- **父承诺**：若此条是某个 goal 的 step，填写父 goal 的 # 编号。`-` 表示无父级。
- **来源对话**：这条承诺是在哪次对话中产生的（`daily/YYYY-MM-DD.md` 或 `long-term.md` 的记录 ID）。
- **前置检查**：到目标日期之前需要做什么？例如「周末看电影」→ 先买票；「下周考公报名」→ 确认报名材料。
- **提醒时间**：Agent 应该在什么 Cron 节点检查这条路？例如 `morning`（晨间）、`night_review`（晚间）、`all`（全天任何节点）。
- **状态**：`pending` = 等待中；`done` = 已完成；`cancelled` = 用户取消了；`suspended` = 用户暂行暂停（需下次 Cron 确认是否恢复）。
```

- [ ] **Step 3: 更新示例数据（第 79-86 行）**

将当前示例表替换为包含新字段的完整示例：

```markdown
### 待办承诺（活跃的）

| # | 类型 | 承诺 | 目标日期 | 父承诺 | 来源对话 | 前置检查 | 提醒时间 | 状态 |
|---|------|------|---------|--------|---------|---------|---------|------|
| 1 | plan | 「周末要去看《哪吒》」 | 2026-08-02 | - | daily/2026-07-30.md | 提醒买票（周三前） | morning | pending |
| 2 | goal | 备考公务员 | 2026-09-15 | - | daily/2026-07-28.md | 确认报名时间 | morning | pending |
| 2.1 | step | 买教材+参考资料 | 2026-08-02 | #2 | daily/2026-07-28.md | - | morning | pending |
| 2.2 | step | 每天刷 30 道行测题 | 每日 | #2 | daily/2026-07-28.md | - | all | pending |
| 2.3 | step | 整理申论素材 | 2026-08-10 | #2 | daily/2026-07-28.md | - | morning | pending |
| 3 | rest_day | 南京旅游（全天外出） | 2026-08-08 | - | 2026-07-30 邮件 | - | - | pending |
| 4 | goal | 开发副业 SaaS | 2026-10-01 | - | daily/2026-07-29.md | 选定技术栈 | morning | pending |
| 4.1 | step | 搭建项目骨架 | 2026-08-03 | #4 | daily/2026-07-29.md | - | morning | pending |
```

- [ ] **Step 4: 更新 CHECK 阶段筛选逻辑（第 44-50 行）**

在第 50 行之后追加新的筛选逻辑：

```markdown
4. 若 Cron 节点为休息日 10:00 晨间启动：
   - 额外筛选目标日期 == "每日" 的 step（每日习惯类提醒）
   - 额外筛选目标日期在未来 7 天内的 step（预览性提醒）
5. 若匹配到 rest_day 声明（类型 == "rest_day" AND 目标日期 == today）：
   → 当前 Cron 节点切换为纯休息模式，仅做日程事件提醒，不做任务安排
```

- [ ] **Step 5: 更新 ACT 阶段写入说明（第 53-67 行）**

在第 67 行之后追加：

```markdown
3. 若意图为全天休息声明（用户说「周六去 ___」/ 「请年假」等）：
   - 类型设为 `rest_day`
   - 目标日期设为用户声明的日期
   - 父承诺、前置检查、提醒时间留空（填 `-`）
   - 承诺字段填入用户原话摘要（如「南京旅游（全天外出）」）
```

- [ ] **Step 6: 新增自动归档规则**

在「LEARN 阶段」章节后追加新的章节：

```markdown
### 自动归档规则

为避免 commitments.md 活跃行膨胀、占用上下文，Agent 在每个 Cron 节点 CHECK 阶段执行以下清理：

1. **日常清理**：将「已完成/取消」区域中 **超过 14 天** 的记录自动移到 `planning/commitments-archive.md`（按月份分组）。

2. **过期清理**（每日 22:30 节点执行）：
   - 扫描「待办承诺」区域：目标日期超过 30 天且状态仍为 `pending` 的记录
   - 在 Cron 邮件中轻量确认一次：「___ 的目标日期已经过了，是延期还是取消？」
   - 若用户 3 天内未回复 → 自动标记 `cancelled`，进入归档流程

3. **容量目标**：确保活跃行 ≤ 20 行。若超过，将最早的 `done`/`cancelled` 记录优先归档。

4. **归档文件**：`planning/commitments-archive.md`，结构：
   - 按月份分组（`## YYYY-MM`）
   - 每条记录包含：#、承诺、目标日期、结果、归档日期
   - 归档后不再参与每日 CHECK，仅在用户主动查询历史时读取
```

- [ ] **Step 7: 更新已完成/取消示例表（第 87-92 行）**

追加一条 rest_day 类型已完成记录示例：

```markdown
| 3 | 南京旅游 | 2026-08-08 | 完成——玩得很开心 | 2026-08-09 |
```

- [ ] **Step 8: Commit**

```bash
git add templates/planning/commitments.md
git commit -m "feat: extend commitments template with type, parent fields, goal/step hierarchy, and archive rules"
```

---

### Task 3: 新建 commitments-archive.md

**Files:**
- Create: `templates/planning/commitments-archive.md`

**Interfaces:**
- Consumes: commitments.md 的归档规则
- Produces: 历史承诺按月归档的存储文件

- [ ] **Step 1: 创建归档文件**

```markdown
# 承诺归档

> 本文件存储已完成的、已取消的、已过期的历史承诺。由 `commitments.md` 的自动归档规则自动移入，不参与每日 CHECK。
>
> **归档规则**：状态变成 `done`/`cancelled` 后超过 14 天，或目标日期超过 30 天且未收到用户回复确认，自动从活跃表移入此处。
>
> **读取策略**：仅在用户主动查询历史承诺或进行月度复盘时读取，不参与 Cron 日常触达。

## 2026-08

| # | 承诺 | 目标日期 | 结果 | 归档日期 |
|---|------|---------|------|---------|
| 1 | 周末去爬山 | 2026-08-02 | 完成 | 2026-08-17 |
| 2 | 报名那个课程 | 2026-08-05 | 取消——用户决定不报了 | 2026-08-20 |

## 2026-07

| # | 承诺 | 目标日期 | 结果 | 归档日期 |
|---|------|---------|------|---------|

## 2026-06

| # | 承诺 | 目标日期 | 结果 | 归档日期 |
|---|------|---------|------|---------|
```

- [ ] **Step 2: Commit**

```bash
git add templates/planning/commitments-archive.md
git commit -m "feat: add commitments archive template for historical tracking"
```

---

### Task 4: 更新 session-flow.md（Phase 4/5/1 适配 rest_day 和三种模式）

**Files:**
- Modify: `engine/session-flow.md`

**Interfaces:**
- Consumes: cron-system.md 的模式定义、commitments.md 的类型和字段
- Produces: 更新后的 session-flow，Phase 4 识别 rest_day 声明，Phase 5 Action E 写入类型和 mode 适配，Phase 1 日间节点跳过逻辑改为三模式判定

- [ ] **Step 1: 更新 Session Context 的 day_type 字段（第 25 行）**

将：

```markdown
| `day_type` | string | `getDayType()` | 当日类型：`workday`、`weekend`、`holiday` |
```

替换为：

```markdown
| `day_type` | string | `getDayType()` | 当日日历类型：`workday`、`weekend`、`holiday` |
| `mode` | string | `determine_mode()` | 最终 Cron 模式：`workday`、`rest_day_default`、`pure_rest` |
```

- [ ] **Step 2: 更新 Phase 1 Step 3 日间节点跳过逻辑（第 107-115 行）**

将旧的假期跳过表替换为三模式判定表：

```markdown
#### Step 3: 日间节点模式判定

根据 `mode`（最终模式，由 `getDayType()` 和 `commitments.md` 中 rest_day 声明共同裁决）决定节点行为。

| `mode` | Cron 槽位类型 | 行为 |
|--------|-------------|------|
| `workday` | 任意 | 正常执行，使用工作日消息模板 |
| `rest_day_default` | 休息日节点（10:00/14:30/17:30） | 正常执行，使用休息日默认消息模板（基于 commitments 提醒） |
| `rest_day_default` | 工作日专属节点（09:30/10:30/15:30/16:30） | 跳过：标记 `phase=skipped_reason=rest_day`，进入 6.闭环 |
| `pure_rest` | 任意休息日节点 | 正常执行，使用纯休息消息模板（仅日程事件，不追任务） |
| `pure_rest` | 工作日专属节点 | 跳过：标记 `phase=skipped_reason=pure_rest`，进入 6.闭环 |
| `workday`（mode 降级） | 任意 | 降级按工作日处理（宁可多触达不可漏触达），但记录 warning 日志 |

**模式判定在 Phase 1 Step 2 之后执行：**

```text
1. cal_type = getDayType(today)
2. user_rest = 读取 commitments.md：
   筛选：目标日期 == today AND 类型 == "rest_day" AND 状态 == "pending"
3. mode = determine_mode(cal_type, user_rest)
4. 将 mode 写入 session_context.mode
```
```

- [ ] **Step 3: 更新 Phase 1 Step 4 读取策略表（第 122-140 行）**

在表末追加新的休息日读取策略行：

```markdown
| `planning/commitments.md` | 跨天承诺追踪：未来意图、目标日期、前置检查 | 按需读取。**休息日节点**：额外筛选 rest_day 声明、目标日期 == "每日" 的 step、未来 7 天到期的里程碑 |
```

并在读取策略列表中追加：

```markdown
- **休息日晨间启动（10:00）**：读取 `user.md`、昨日 `memory/daily/*.md`、`commitments.md`（筛选 rest_day 声明 + 今日到期 step + 每日 step + 未来 7 天里程碑）。不读取天气和日历（休息日不需要）。
- **休息日午后跟进（14:30）**：读取 `user.md`、今日 `memory/daily/*.md`、10:00 触达记录（如果有）。
- **休息日日终收尾（17:30）**：读取 `user.md`、今日 `memory/daily/*.md`、`commitments.md`（检查今日承诺执行状态）。
```

- [ ] **Step 4: 更新 Phase 4 分类 Prompt（第 462-509 行）**

在分类 Prompt 的「分类标准」部分追加第 6 种分类描述：

在 `分类标准：` 部分，`5. 深度问题（deep_question）...` 之后追加：

```text
6. 休息日声明（rest_day_declaration）：用户表达了某天（或某段时间）全天外出/旅游/休息/请假的意图——不是在安排一个承诺，而是在声明某天不可用于工作/推进。例如：「周六去南京旅游」「下周三请假全天在医院」「周日想完全放空」。
```

在「区分 create_intent vs modification_request 的关键」部分之后追加：

```text
区分 create_intent vs rest_day_declaration 的关键：
- 如果用户说「周末要去看电影」→ create_intent（计划去做某件具体的事）
- 如果用户说「周六去南京旅游」→ rest_day_declaration（全天被占用，不是单一承诺）
- 如果用户说「周日想完全放空」→ rest_day_declaration（明确表达全天休息意图）
- 如果用户说「下周请假」→ rest_day_declaration（工作日请假，覆盖原工作模式）
- 边界：如果用户说「周末要去南京参加婚礼」→ 默认归为 create_intent + 额外标记该日为 rest_day？还是只归为 rest_day_declaration？优先归为 rest_day_declaration——全天外出意味着无法做其他事，不需要再问「还想做什么」。
```

在输出格式 JSON 中增加可选字段：

```text
输出格式（JSON）：
{
  "classification": "<六种类型之一>",
  "confidence": <0.0 到 1.0>,
  "summary": "<一句话总结用户说了什么>",
  "key_signals": ["<识别到的关键信号词或短语>"],
  "intent_fields": {
    "what": "<用户想做什么（create_intent）或声明了什么（rest_day_declaration）>",
    "when": "<目标日期/时间范围>",
    "type": "<commitment 类型：plan / goal / rest_day>",
    "prerequisite": "<用户提到的前置条件或检查项，如果有>"
  },
  "reasoning": "<归类依据，引用用户原文中的具体表述>"
}
```

在优先级规则部分追加：

```text
6. **休息声明优先**：如果用户表达了全天外出/休息/请假意图（含日期和时间范围），优先归为 `rest_day_declaration`，即使描述中包含情绪或状态内容。此类声明直接影响 Cron 调度模式，需要立即写入 commitments.md。
```

- [ ] **Step 5: 更新 Phase 5 Action E（第 675-738 行）**

在 Action E 的操作动作中追加 `rest_day` 类型的写入逻辑。

定位第 687-703 行（Action E 操作动作列表），在末尾追加第 5 项：

```markdown
5. 若用户的意图类型为 `rest_day`（全天休息声明）：
   - 写入 `planning/commitments.md`，类型设为 `rest_day`
   - 承诺内容填用户声明摘要（如「南京旅游（全天外出）」）
   - 父承诺、前置检查、提醒时间留空（填 `-`）
   - 若用户声明了日期范围（如「初一到初五在老家」）→ 每个日期创建一条独立记录（便于每日 CHECK 命中）
   - 回复用户确认：「收到，{日期} 标记为休息日了。那天我不会追问任务，但如果有日程事件（如高铁时间）会照常提醒。」
```

在 Action E 的写入数据表（第 706-711 行）中追加一行：

```markdown
| `planning/commitments.md` | 待办承诺（rest_day 类型——日期范围） | 每个独立日期一条记录 |
```

- [ ] **Step 6: 更新边界情况「节假日跳过」章节（第 863-869 行）**

将旧的节假日跳过描述替换为适配三模式的新描述：

```markdown
### 非工作日模式切换

**场景**：Cron 触发日间节点时，`mode`（最终模式）非 `workday`。

**处理**：
1. Phase 1 判定为 `rest_day_default` 或 `pure_rest` 时，工作日专属节点跳过（标记 `skipped_reason=rest_day` 或 `skipped_reason=pure_rest`），共享节点照常执行。
2. 休息日节点（10:00/14:30/17:30）根据 mode 选择对应模板。
3. 晚间 22:30 节点照常执行（三种模式的消息模板不同）。
4. 连续假日抑制规则见 `engine/cron-system.md` 的「假期连续提醒抑制」章节。
```

- [ ] **Step 7: 提交**

```bash
git add engine/session-flow.md
git commit -m "feat: adapt session-flow to three-mode scheduling with rest_day detection and routing"
```

---

### Task 5: 更新 algorithm.md（CHECK/GAP 适配三种模式）

**Files:**
- Modify: `system/algorithm.md`

**Interfaces:**
- Consumes: cron-system.md 的三模式定义，session-flow.md 的 mode 字段
- Produces: 更新后的 algorithm，CHECK 步骤适配三模式差异，GAP 步骤新增差距类型

- [ ] **Step 1: 更新 CHECK 读取表中的 Cron 触达场景（第 111 行）**

将：

```markdown
| Cron 触达（早安/状态检查/日终/晚间） | `profile/user.md`、`memory/daily/{today}.md`、`memory/daily/{yesterday}.md`、`planning/commitments.md`、外部日历 API、外部待办 API |
```

替换为：

```markdown
| Cron 触达（工作模式） | `profile/user.md`、`memory/daily/{today}.md`、`memory/daily/{yesterday}.md`、`planning/commitments.md`、`planning/daily-plan.md`、外部日历 API、外部待办 API |
| Cron 触达（休息日默认模式） | `profile/user.md`、`memory/daily/{yesterday}.md`、`planning/commitments.md`（筛选 rest_day 声明 + 今日到期/每日 step + 未来 7 天里程碑）、不读取 daily-plan.md（休息日不写强日计划） |
| Cron 触达（纯休息模式） | `profile/user.md`、`memory/daily/{yesterday}.md`、`planning/commitments.md`（仅筛选 rest_day 声明和日程事件类承诺）、不读取 daily-plan.md |
```

- [ ] **Step 2: 更新 CHECK 输出摘要（第 124-130 行）**

在「时间上下文」行追加 mode：

```markdown
   - 时间上下文：当前时间点、日类型（workday/weekend/holiday）、**最终模式（workday/rest_day_default/pure_rest）**。
```

- [ ] **Step 3: 更新 GAP 差距类型表（第 152-161 行）**

在差距类型表末尾追加一行：

```markdown
| `commitment_overdue` | 承诺或长期目标 step 已过期或即将到期 | commitments.md 中目标日期在今天或之前的 pending 项 |
| `rest_day_override` | 用户声明了工作日为纯休息（请年假等），原工作日计划需调整 | 当天有 rest_day 声明且当日 mode 为 pure_rest |
| `rest_day_step_available` | 休息日默认模式下有今日到期或「每日」step 可推进 | commitments.md 中目标日期 == today 或 == "每日" 的 step |
```

- [ ] **Step 4: 更新 DECIDE 匹配动作表（第 196-204 行）**

在动作匹配表末尾追加：

```markdown
| `rest_day_override` | `plan_adjustment` | 工作日被 rest_day 覆盖，将该日的原计划 task 和 commitment 关联起来——询问用户是延期还是取消 |
| `rest_day_step_available` | `gentle_nudge` | 休息日默认模式，轻量提醒有可推进的 step——不追进度，提供跳过选项 |
```

- [ ] **Step 5: Commit**

```bash
git add system/algorithm.md
git commit -m "feat: adapt algorithm CHECK and GAP steps for three-mode scheduling"
```

---

### Task 6: 更新 AGENTS.md 路由表

**Files:**
- Modify: `AGENTS.md`

**Interfaces:**
- Consumes: 新增的 rest_day_declaration 分类、rest_day 数据存储、三模式 Cron 行为
- Produces: 更新后的 AGENTS.md，路由表涵盖新分类和模式

- [ ] **Step 1: 更新记忆场景列表中的休息日描述**

在 AGENTS.md 中找到关于休息日行为的描述，将「休息日不追任务」更新为「休息日默认基于长期目标提醒」。

- [ ] **Step 2: 在四层路由表中追加新行**

在路由表中追加：

```markdown
| 用户表达了全天休息/外出/请假意图 | `rest_day_declaration` | Phase 5 Action E → 写入 commitments.md（类型=rest_day） | `engine/session-flow.md` Phase 4/5, `templates/planning/commitments.md` |
| Cron 触发时需判断今日模式 | `determine_mode()` | Phase 1 Step 2→3 → `mode` 写入 session_context | `engine/cron-system.md` 非工作日模式章节, `engine/session-flow.md` Phase 1 Step 3 |
```

- [ ] **Step 3: 更新最低执行协议 #1 预读清单**

在 `planning/commitments.md` 的说明中追加：`（含 rest_day 声明和三模式判定）`

- [ ] **Step 4: Commit**

```bash
git add AGENTS.md
git commit -m "feat: wire rest_day routing into AGENTS.md entry points and layered routing table"
```

---

### Task 7: 全局验证

**Files:**
- 验证：所有修改后的文件一致性

- [ ] **Step 1: 检查跨文件一致性**

```bash
# 确认所有文件引用的字段名一致
grep -n "rest_day\|纯休息\|休息日默认\|rest_day_default\|pure_rest\|父承诺\|goal\|step" \
  engine/cron-system.md \
  engine/session-flow.md \
  system/algorithm.md \
  templates/planning/commitments.md \
  AGENTS.md
```

手动检查输出：确保 `rest_day` / `rest_day_declaration` / `rest_day_default` / `pure_rest` / `mode` 在不同文件中的语义一致。

- [ ] **Step 2: 检查 cron-system.md 节点定义和 session-flow.md 跳过逻辑的对齐**

确认：
- cron-system.md 中标记「仅工作日」的节点与 session-flow.md Phase 1 Step 3 中的跳过逻辑一致
- 休息日节点的时间标签在 cron-system.md 和 session-flow.md 中一致

- [ ] **Step 3: 检查 commitments.md 新字段在所有引用处的一致性**

```bash
grep -c "类型\|父承诺\|rest_day\|goal\|step\|每日" templates/planning/commitments.md
```

- [ ] **Step 4: Commit（如有一致性修复）**

```bash
git add -A
git commit -m "chore: cross-file consistency fixes for rest day redesign"
```
