### 任务规划流程

| 流程 | 状态 | 详情 |
|------|------|------|
| ~~~ 触发任务规划 ~~~ | — | — |
| 用户在邮件/对话中表达「我打算下个月考公」 | ⚠️ 部分完整 | session-flow.md Phase 4 分类表新增了 create_intent（新建意图/计划）标签。分类 Prompt 中包含 intent_fields 用于提取意图内容、时间和前置条件。 |
| 用户对已有计划说「考公推迟一个月」 | ✅ 完整 | Phase 4 分类 modification_request（修改请求）描述中包含「之前说的考公推迟一个月」作为示例。Phase 5 Action C 操作表中包含「未来承诺修改（延期/取消/改日期）」行。 |
| 用户说「不确定要不要考公，值不值得」 | ⚠️ 部分完整 | 分类 Prompt 中有一条区分规则提及「不确定这事儿值不值得列入计划 → 归为 deep_question 而非 create_intent」。但 AGENTS.md 的四层路由表中缺少对应的路由条目。 |
| ~~~ 存储未来承诺 ~~~ | — | — |
| 清晰意图 → 写入 commitments.md | ✅ 完整 | Phase 5 Action E（create_intent）操作动作 #1：直接写入 planning/commitments.md，附带 #ID、目标日期、来源、前置检查、提醒时间。 |
| 模糊意图 → 存入 long-term.md 待确认 | ✅ 完整 | Action E 操作动作 #2：标记为「待确认意图」，下次 Cron 触达时由 Agent 读取提醒。 |
| 长期承诺 → 创建 ISA 文档 | ✅ 完整 | Action E 操作动作 #3：如意图足够大则创建 ISA 文档（引用 system/isa-system.md）。非承诺类项目写入 projects/projects.md。 |
| 前置动作 → 作为关联承诺独立存储 | ✅ 完整 | Action E 操作动作 #4：如「周末看电影」→ 拆分出「为看电影买票」作为关联承诺。 |
| ~~~ Cron 读取路径 ~~~ | — | — |
| 引擎层：session-flow.md Phase 1 收集用户上下文 | ✅ 完整 | 读取表中新增 commitments.md 条目；读取策略表更新：所有 6 个 Cron 槽位均包含 commitments.md，标注了各自读取原因。 |
| 系统层：algorithm.md CHECK 步骤读取列表 | ✅ 完整 | CHECK 读取表中新增 commitments.md |
| 系统层：algorithm.md GAP 步骤预期状态来源 | ✅ 完整 | GAP 部分的预期状态来源中新增 commitments.md |
| 系统层：algorithm.md GAP 差距类型表 | ✅ 完整 | 差距类型表新增了 commitment-overdue 行，映射到 commitment-followup 动作。 |
| 引擎层：cron-system.md 各时间点表格 | ✅ 完整 | 时间点定义表更新：每个定时节点（09:30 / 10:30 / 14:30 / 15:30 / 16:30 / 17:30 / 22:30）的回顾和展望列均包含 commitment 检查。 |
| 引擎层：cron-system.md 回顾规则 | ✅ 完整 | 基于文件回顾的规则中新增 commitments.md |
| ~~~ AGENTS.md 入口点集成 ~~~ | — | — |
| AGENTS.md 记忆场景列表 | ✅ 完整 | 新增项目符号：「用户表达了未来要做某事的意图…→ 读取 planning/commitments.md」 |
| AGENTS.md 最低执行协议 #1 | ✅ 完整 | 做日程/计划前的预读清单中新增 commitments.md：「优先读取 … planning/commitments.md …」 |
| AGENTS.md 最低执行协议 #5 | ✅ 完整 | 「未来意图和跨天承诺写入 planning/commitments.md」区分：短期任务（dida）/项目（projects.md）/未来意图（commitments.md）|
| AGENTS.md 四层路由表 | ❌ 缺失 | 路由表中没有新增行来将「用户表达未来意图 / 创建新计划 / 承诺追踪」与 create_intent / Action E / commitments.md 机制连接起来。 |

### 休息日规划

| 方面 | 状态 | 详情 |
|------|------|------|
| Cron 休息日跳过逻辑 | ✅ 完整 | cron-system.md 第 109-113 行：休息日日间节点跳过，仅 22:30 触发（使用轻量消息模板）。 |
| 休息日晚间复盘提及承诺 | ✅ 完整 | 22:30 休息日展望列更新为包含：轻量提示未来一周的承诺概览（「下周有个 ___，有没有需要提前准备的？」）。 |
| 休息日日间忘记检查承诺 | ❌ 缺失 | 仅在 22:30 提及承诺——但每月 09:30 仍然是唯一执行完整承诺检查的槽位。 |

### 已交付（4 个提交）

1. `07bea70` — 基础 create_intent + commitments 模板 + Action E + Action C 增强 + 清理待编写标注
2. `5b48964` — 接入：引擎层（session-flow READ 表、algorithm READ 表、GAP 来源、GAP 类型、cron-system 时间点表、回顾规则）
3. `2d87bf5` — 入口：AGENTS.md 记忆场景、先读清单、写协议
4. `ca0bec4` — 补充：AGENTS.md 路由表用户意图场景

### 缺口

1. **休息日日间承诺检查**：22:30 的轻量提及没问题，但如果用户在周末白天需要对承诺采取行动，目前没有任何检查。
2. **没有事前提醒语义**：commitments.md 示例显示 #1「看电影」目标日期为 2026-08-02（周日），状态 pending。但工作流只说「目标日期时提醒」——没有「距离目标日期还有 5 天，该买票了」的逻辑。10:30 和 14:30 的更新描述了前置条件检查，但未在 commitments.md 字段中体现。
3. **深度问题与意图模糊的边界**：分类 Prompt 中有一条规则：如果用户不确定某事是否值得列入计划 → 归为 deep_question，而非 create_intent。但 AGENTS.md 路由表中缺少对应行。

### 尚有问题（待解决）

- cron-system.md 中，休息日日间跳过 + 晚间轻量提及之间的紧张关系：如果用户仅靠 22:30 的承诺提醒，他们直到睡前一小时才知道明天有承诺到期。
