# 晚间节点 + 承诺优先填充 + 邮件→滴答自动写入 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 新增晚间三节点（20:00/21:00/22:00），全 Cron 节点统一从「询问」改为「承诺优先填充」，邮件中的意图自动创建为滴答任务，复盘从 22:30 迁移至 23:00。

**Architecture:** 在现有 engine/cron-system.md、engine/session-flow.md、system/algorithm.md、integrations/dida-mcp.md 和 commitments 模板上做增量改动。不新建文件，不改核心框架。晚间节点遵循白天节点相同的「回顾→开启」模式，语气更轻、不加动态 Timer。承诺优先填充统一为「多数据源拉取→自动填充→呈现确认→同步滴答」四步流水线。

**Tech Stack:** 纯 Markdown 规格文档修改，无代码实现。外部依赖：企微邮件 API（触达通道）、滴答 MCP（任务创建）、timor.tech API（工作日判断）。

## 全局约束

- 晚间三节点（E1/E2/E3）仅工作日运行，休息日不加块
- 晚间节点无 +5/+10 min 动态 Timer
- 22:30 复盘时间改为 23:00，内容完整迁移，休息日轻量复盘照常
- 滴答 MCP 不可用时降级为本地 commitments.md + `[待同步到滴答]` 标记
- 滴答创建任务前必须先查今日任务列表做幂等检查（标题匹配跳过）
- commitments.md step 表新增「类别」列为可选字段
- 全节点开启动作统一为承诺优先填充，用户不回按填充默认执行
- 节点总数从 11 个变为 14 个

---

### Task 1: 更新 cron-system.md 时间点定义表（新增晚间三节点 + 23:00 迁移）

**Files:**
- Modify: `engine/cron-system.md`

**Interfaces:**
- Consumes: 无（首个改动）
- Produces: 14 个节点的完整时间点定义表，E1/E2/E3 节点定义，23:00 节点定义（替代原 22:30），各节点回顾/开启字段

- [ ] **Step 1: 替换时间点定义表**

将现有表（18-31行）替换为包含 E1/E2/E3 和 23:00 的新表。原 #7（22:30）改为 N（23:00），原 R1/R2/R3 编号不变，D1/D2（+5min/+10min）编号不变。新增 E1/E2/E3 三个行。

```markdown
| # | 时间点 | 触发方式 | 运行条件 | 回顾（Review） | 开启（Look Ahead） | 备注 |
|---|--------|---------|---------|-------------|----------------|------|
| 1 | **09:30** | Fixed Cron | 仅工作日 | 回顾昨日 `memory/daily/{yesterday}.md` 的复盘结论，若有未完成事项标记为「今天继续」；检查 `planning/commitments.md` 中今日到期及近日目标日期的承诺 | 早安问候 + 今日计划引导。从多数据源拉取候选动作（commitments 今日到期 + 昨日复盘未完成 + 长期目标下一步 + 滴答今日任务），自动填充今日计划草案，呈现给用户确认 | 一天的第一个节点；全节点承诺优先填充的首个执行点 |
| 2 | **10:30** | Fixed Cron | 仅工作日 | 回顾 09:30 计划的实际执行情况：距离今日计划过去了 1 小时 | 上午状态检查：进度、精力、需调整？若有 commitments 中已过期的前置检查项或 09:30 填充计划当前段未推进项，轻量提醒 | 不重排计划，仅基于 09:30 既定计划做提醒 |
| 3 | **14:30** | Fixed Cron | 仅工作日 | 回顾上午整体执行：上午计划 vs 实际完成率；精力线索 | 午后启动引导：从 commitments 拉取本周末到期前置条件 + 上午未完成项延续，自动填充下午 1-2 件重点，呈现确认 | 用上午实际数据校准下午安排 |
| 4 | **15:30** | Fixed Cron | 仅工作日 | 回顾 14:30-15:30 执行：14:30 确定的下午重点推进了吗 | 下午中段状态检查：进度和精力，需切换任务类型？若 14:30 事项卡住则建议切换 | 不让用户在同一任务上卡整个下午 |
| 5 | **16:30** | Fixed Cron | 仅工作日 | 回顾 15:30-16:30 执行 + 全天累计完成率 | 收尾预备：检查 commitments 今日到期但全天未提及的漏网之鱼（自动抓漏），列出用户未主动推进的到期承诺 | 一天最后一个完整执行窗口的抓漏节点 |
| 6 | **17:30** | Fixed Cron | 工作日 + 休息日 | 若工作日：回顾全天执行全貌、精力曲线、情绪模式。若休息日：回顾承诺执行情况 | 若工作日：未完成项处置（延期/取消/明天）；明天最重要 1 件事。若休息日：未完成项处置、次日衔接 | 收尾为主，不主动填充新计划 |
| **E1** | **20:00** | Fixed Cron | 仅工作日 | 回顾今日白天执行摘要（17:30 收尾结论）+ 近 7 天晚间四类覆盖快照（从 memory/daily/ 近 7 天晚间段统计精进/休闲/锻炼/副业各出现次数） | 开启晚间第一块：第一层从 commitments 四类标签 + 长期目标晚间可执行步骤 + 习惯缺口 + 邮件提取意图拉取候选；第二层用覆盖缺口兜底。填充三块呈现确认 | 晚间节点语气更轻、不追进度、无动态 Timer；用户不回按填充默认执行 |
| **E2** | **21:00** | Fixed Cron | 仅工作日 | 回顾第一块实际做了什么，是否按计划推进；自动判定第一块归属类别 | 开启第二块：按第一块已覆盖类别调整建议，切换还是继续 | 若用户未回 E1，E2 检查是否有新回复；仍无回复按 E1 填充默认继续 |
| **E3** | **22:00** | Fixed Cron | 仅工作日 | 回顾第一、二块执行；自动判定归属类别 | 开启第三块：晚间最后一小时，该收束的收束。若前两块已覆盖精进+锻炼，第三块建议休闲 | 晚间最后一个执行块 |
| **N** | **23:00** | Fixed Cron | 每日 | 回顾晚间三块（仅工作日）+ 全天执行全貌；若纯休息则回顾当日体验亮点 | 次日预览：若次日为工作日提示查看日程/待办/commitments；若休息日轻量提示下周承诺概览 | 原 22:30 复盘内容完整迁移至 23:00；休息日轻量复盘照常 |
| R1 | **10:00** | Fixed Cron | 仅休息日/节假日 | 读取昨日 daily 和 commitments.md，筛选今日到期或标记「每日」的 step，以及未来 7 天里程碑 | 休息日晨间启动：主动列出今日相关承诺和长期目标拆解；若命中 rest_day 声明切换为纯休息模式 | 比工作日晚 30 分钟 |
| R2 | **14:30** | Fixed Cron | 仅休息日/节假日 | 检查上午是否有用户回复或执行记录；回顾 10:00 列出的事项 | 若用户已推进 → 肯定 + 问下午计划；若未回复 → 轻量提醒不催促；若已说不做 → 记录为纯休息 | 与工作日共享 14:30 槽位但内容不同 |
| R3 | **17:30** | Fixed Cron | 仅休息日/节假日 | （同 #6 休息日部分） | （同 #6 休息日部分） | 与工作日共享 17:30 槽位 |
| 10 | **发送后+5min** | Dynamic Timer | 动态（发送后） | 检查用户是否回复了刚才的发送 | 未回复 → 温和提醒；已回复 → 取消 Timer | 晚间节点（E1/E2/E3）不触发动态 Timer |
| 11 | **发送后+10min** | Dynamic Timer | 动态（发送后） | 再次检查用户是否回复 | 归档为「等待用户回复」 | 同上 |
```

- [ ] **Step 2: 更新回顾规则章节，新增晚间类别判定规则**

在现有的「回顾规则（通用）」章节末尾（第 4 条之后）追加第 5 条：

```markdown
5. **晚间块的类别自动判定**：E2 和 E3 的回顾阶段，系统根据用户回复中的关键词自动判定上一个晚间块归属的类别：
   - 包含「看/读/学/纪录片/课程/练习」→ 精进
   - 包含「刷剧/游戏/休息/放松/躺/随便」→ 休闲
   - 包含「跑/健身/运动/走/跳/游泳」→ 锻炼
   - 包含「副业/项目/赛道/搞钱/研究」→ 副业
   判定结果写入当日 daily 的「晚间」段，供次日 E1 统计覆盖快照。
```

- [ ] **Step 3: 更新时间点设计原理章节**

在「休息日缺少 10:30/15:30/16:30」说明之后追加晚间节点设计原理：

```markdown
- **20:00**：大多数用户已下班回家、吃完晚饭，适合开启晚间个人时间的第一块。
- **21:00**：晚间第一块通常耗时 1 小时，第二块开始前做一次轻量切换检查。
- **22:00**：距离入睡约 1-2 小时，晚间最后一个执行窗口，适合收束。
- **23:00**：睡前约 30-60 分钟，适合全天复盘而不影响入睡。
```

- [ ] **Step 4: 晚间节点行为章节 22:30 → 23:00**

将「晚间节点行为」章节（335 行起）中所有「22:30」替换为「23:00」。工作日晚间复盘内容中新增「晚间三块回顾」：

```markdown
**工作日晚间复盘（23:00）：**

回顾晚间三块的实际执行情况（从 E1/E2/E3 节点的记录中汇总），在复盘邮件中轻量提及覆盖分布。不强制评价类别均衡性——只呈现数据，让用户自己感受。
```

- [ ] **Step 5: 更新 Cron 表达式参考表**

将原 22:30 行改为 23:00，新增 E1/E2/E3 三行。

- [ ] **Step 6: 更新用户可参数化配置**

将 `cron.night_review_time` 默认值从 `22:30` 改为 `23:00`。新增：

```markdown
| `cron.evening_block1_time` | `20:00` | 晚间第一块启动时间。设为 `""` 可禁用晚间所有节点。 |
| `cron.evening_block2_time` | `21:00` | 晚间第二块时间。 |
| `cron.evening_block3_time` | `22:00` | 晚间第三块时间。 |
```

- [ ] **Step 7: 更新 /loop 命令和 hooks 配置示例**

将所有 `22:30` 替换为 `23:00`。新增 E1/E2/E3 的 `/loop` 示例。

- [ ] **Step 8: Commit**

```bash
git add engine/cron-system.md
git commit -m "feat: add evening nodes E1/E2/E3, migrate 22:30 review to 23:00, add category tracking"
```

---

### Task 2: 更新 cron-system.md 全节点承诺优先填充 + 四类覆盖追踪

**Files:**
- Modify: `engine/cron-system.md`

**Interfaces:**
- Consumes: Task 1 的新节点表
- Produces: 承诺优先填充的通用机制描述、各节点填充数据源对照表、晚间四类覆盖追踪机制、14 节点总览

- [ ] **Step 1: 新增「承诺优先填充机制」章节**

在「回顾规则（通用）」之后、「时间点设计原理」之前新增。内容包含：核心原则（5 步流水线）、数据源优先级表、各节点填充来源对照表。

- [ ] **Step 2: 新增「晚间四类覆盖追踪」章节**

在「休息日参数化配置」之后新增。内容包含：四类定义与关键词、追踪方式（E2/E3 自动判定 → daily 记录 → E1 读取近 7 天统计 → 缺口建议）。

- [ ] **Step 3: 新增「节点总览（14 个）」章节**

在维护说明之前新增节点总览表（工作日 10 个 + 休息日 4 个）。

- [ ] **Step 4: Commit**

```bash
git add engine/cron-system.md
git commit -m "feat: add commitment-first auto-fill mechanism, evening category tracking, 14-node overview"
```

---

### Task 3: 更新 session-flow.md Phase 4 新增 task_commitment 分类 + Phase 5 Action G

**Files:**
- Modify: `engine/session-flow.md`

**Interfaces:**
- Consumes: 现有 Phase 4 六种分类体系
- Produces: 新增 `task_commitment` 分类（第七种），含匹配模式、推断规则、Action G 完整定义

- [ ] **Step 1: 更新 Phase 4 分类体系表**

在分类体系表中新增 `task_commitment` 行（第 7 种类型）。

- [ ] **Step 2: 更新分类 Prompt**

在分类 Prompt 中新增 task_commitment 的描述和与 create_intent 的区分标准。

- [ ] **Step 3: 新增 task_commitment 匹配模式与推断规则表**

新增章节列出 5 种触发模式（「今天要 X」「明天记得 X」「周末 X」「下次 X」「得 X 一下」）及对应的 due_date/priority 自动推断。

- [ ] **Step 4: 新增 Action G: task_commitment**

在 Phase 5 新增 Action G，包含：目标、承接方式、操作动作（幂等检查 → 创建滴答任务 → 写入 commitments.md 备份 → 回复用户确认）、回复邮件模板。

- [ ] **Step 5: 更新 Action 执行判断表**

新增 Action G 行。

- [ ] **Step 6: Commit**

```bash
git add engine/session-flow.md
git commit -m "feat: add task_commitment classification and Action G for email-to-dida task creation"
```

---

### Task 4: 更新 session-flow.md Phase 1 晚间节点上下文收集

**Files:**
- Modify: `engine/session-flow.md`

**Interfaces:**
- Consumes: Phase 1 Step 4 现有读取策略表、Step 3 模式判定表
- Produces: E1/E2/E3 节点的上下文读取策略、晚间节点模式判定规则

- [ ] **Step 1: 更新 Phase 1 Step 4 读取策略表**

新增 E1（20:00）/ E2（21:00）/ E3（22:00）的读取策略行。将原「晚间复盘（22:30）」改为「晚间复盘（23:00）」。

- [ ] **Step 2: 更新 Phase 1 Step 3 模式判定表**

新增晚间节点行：workday 模式下晚间节点正常执行（使用晚间消息模板），rest_day_default/pure_rest 模式下跳过。

- [ ] **Step 3: Commit**

```bash
git add engine/session-flow.md
git commit -m "feat: add evening node context collection rules to Phase 1"
```

---

### Task 5: 更新 algorithm.md CHECK 步骤 + GAP 晚间类型 + LEARN 类别判定

**Files:**
- Modify: `system/algorithm.md`

**Interfaces:**
- Consumes: 现有 CHECK → GAP → DECIDE → ACT → VERIFY → LEARN 六步循环
- Produces: CHECK 步骤新增「多数据源拉取候选动作」子步骤和「晚间四类覆盖统计」子步骤；GAP 新增 `evening_category_gap` 和 `task_commitment_detected` 差距类型；LEARN 新增晚间类别判定规则

- [ ] **Step 1: CHECK「做什么」新增第 5 项**

「拉取候选动作（承诺优先填充）」：从 commitments/昨日复盘/TELOS/滴答/习惯缺口/邮件意图按优先级拉取。

- [ ] **Step 2: CHECK「做什么」新增第 6 项（E1 专用）**

「晚间四类覆盖统计」：读近 7 天 daily 晚间段统计四类覆盖，标记连续缺失。

- [ ] **Step 3: GAP 差距类型表新增 2 行**

`evening_category_gap` 和 `task_commitment_detected`。

- [ ] **Step 4: LEARN 新增「晚间类别判定」规则**

复盘时按关键词自动判定每个晚间块归属类别并写入 daily。

- [ ] **Step 5: CHECK「按需读取」表新增 E1/E2/E3 行**

- [ ] **Step 6: Commit**

```bash
git add system/algorithm.md
git commit -m "feat: add candidate action pulling to CHECK, evening gap types, category auto-tagging"
```

---

### Task 6: 更新 dida-mcp.md 自动写入规范

**Files:**
- Modify: `integrations/dida-mcp.md`

**Interfaces:**
- Consumes: 现有能力映射表、降级策略
- Produces: 新增「自动写入规范」章节（触发场景表、幂等检查流程、推断规则表）

- [ ] **Step 1: 新增「自动写入规范」章节**

在「能力映射表」之后新增。包含：3 种触发场景（Cron 填充/邮件意图/用户确认）、幂等检查流程（get_today_tasks → 标题模糊匹配 → 跳过或创建）、推断规则表（时间词 → due_date、紧迫词 → priority、内容 → tags）。

- [ ] **Step 2: 更新调用频次限制表**

新增自动写入场景的行。

- [ ] **Step 3: Commit**

```bash
git add integrations/dida-mcp.md
git commit -m "feat: add auto-write spec, idempotency check, and inference rules to dida-mcp integration"
```

---

### Task 7: 更新 commitments.md 模板（新增类别列）

**Files:**
- Modify: `templates/planning/commitments.md`

**Interfaces:**
- Consumes: 现有 8 列表头
- Produces: 新增「类别」列（可选），更新字段说明、示例数据、CHECK 筛选逻辑

- [ ] **Step 1: 待办承诺表头新增「类别」列**

`| # | 类型 | 承诺 | 目标日期 | 父承诺 | 类别 | 来源对话 | 前置检查 | 提醒时间 | 状态 |`

- [ ] **Step 2: 字段说明新增类别字段**

`类别`：可选值：精进/休闲/锻炼/副业。不填时晚间系统根据关键词实时判定。

- [ ] **Step 3: 示例数据行添加类别标签**

为晚间相关条目补充类别（如 step 2.2 添加「精进」、step 4.1 添加「副业」）。

- [ ] **Step 4: CHECK 阶段说明新增 E1 筛选逻辑**

E1 节点额外筛选四类标签的 pending step/goal 和长期目标晚间可执行步骤。

- [ ] **Step 5: Commit**

```bash
git add templates/planning/commitments.md
git commit -m "feat: add optional category column to commitments template for evening use"
```

---

### Task 8: 最终验证与收尾

**Files:**
- 验证: 所有上述 5 个文件

**Interfaces:**
- Consumes: Task 1-7 所有产出
- Produces: 验证通过的完整文档集

- [ ] **Step 1: 验证 cron-system.md 14 个时间点全覆盖**

```bash
grep -c "09:30\|10:30\|14:30\|15:30\|16:30\|17:30\|20:00\|21:00\|22:00\|23:00" engine/cron-system.md
```

- [ ] **Step 2: 验证无 TBD/占位符**

```bash
grep -rn "TBD\|TODO\|FIXME\|待补充\|待定" engine/cron-system.md engine/session-flow.md system/algorithm.md integrations/dida-mcp.md templates/planning/commitments.md
```
预期：无匹配。

- [ ] **Step 3: 验证 task_commitment 分类完整**

```bash
grep -c "task_commitment" engine/session-flow.md
```
预期：至少 3。

- [ ] **Step 4: 验证 22:30 已全部替换为 23:00**

```bash
grep -n "22:30" engine/cron-system.md engine/session-flow.md
```
预期：仅在迁移注释中出现。

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "chore: final verification pass for evening nodes and auto-fill implementation"
```
