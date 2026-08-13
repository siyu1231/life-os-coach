# 滴答清单 MCP 集成

> 定义 Agent 如何通过 MCP 协议与滴答清单（Dida）交互，包括连接配置、能力映射、降级策略、数据边界和操作规范。

## 零、MCP 连接配置

### 服务器信息

| 项目 | 值 |
|------|-----|
| **MCP 服务器 URL** | `https://mcp.dida365.com` |
| **传输协议** | Streamable HTTP |
| **认证方式** | OAuth 2.0（推荐）或 Bearer Token |

### Bearer Token 获取方式

如需使用 Bearer Token 替代 OAuth，前往 [网页版滴答清单](https://dida365.com)，点击「头像」→「设置」→「账户与安全」→「API 口令」，创建并复制 Token。

### 各 AI 工具配置方法

#### Claude Desktop

1. 打开 Claude Desktop，进入 **Customize** > **Connectors**
2. 点击 "+"，选择 **Add Connector**
3. 填写 MCP 服务器 URL：`https://mcp.dida365.com`
4. 保存后点击 **Connect**，按提示完成 OAuth 登录和授权

#### Claude Code

OAuth 方式（推荐）：

```bash
claude mcp add --transport http dida365 https://mcp.dida365.com
```

然后在 Claude Code 会话中运行 `/mcp`，按提示完成 OAuth 授权。

Bearer Token 方式：

```bash
claude mcp add --transport http dida365 https://mcp.dida365.com --header "Authorization: Bearer YOUR_TOKEN_HERE"
```

#### Cursor

1. 打开 Cursor，进入 **Cursor Settings** > **Tools & MCP** > **Add Custom MCP**
2. 编辑 `.cursor/mcp.json`：

```json
{
  "mcpServers": {
    "dida365": {
      "url": "https://mcp.dida365.com"
    }
  }
}
```

3. 保存后在已安装的 MCP 服务中找到滴答清单，点击 **connect** 完成 OAuth 授权

Bearer Token 方式：

```json
{
  "mcpServers": {
    "dida365": {
      "url": "https://mcp.dida365.com",
      "headers": {
        "Authorization": "Bearer YOUR_TOKEN_HERE"
      }
    }
  }
}
```

#### VS Code

1. 创建工作区文件 `.vscode/mcp.json`：

```json
{
  "servers": {
    "dida365": {
      "type": "http",
      "url": "https://mcp.dida365.com"
    }
  }
}
```

2. 保存后按指引在浏览器中完成 OAuth 授权
3. 也可通过命令面板（Ctrl+Shift+P）运行 **Add Server**，选择 **HTTP**，输入 URL 和 ID

Bearer Token 方式在配置中添加 `headers` 字段：

```json
{
  "servers": {
    "dida365": {
      "type": "http",
      "url": "https://mcp.dida365.com",
      "headers": {
        "Authorization": "Bearer YOUR_TOKEN_HERE"
      }
    }
  }
}
```

#### Codex

终端命令方式：

```bash
codex mcp add dida365 --url https://mcp.dida365.com
```

或通过 Codex App：**设置 > 插件 > MCP > 添加服务器**，选择「流式 HTTP」，输入 URL，保存后点击「进行身份验证」。

#### ChatGPT

支持 Business 及 Enterprise 套餐。进入 **设置 > 应用 > 高级设置**，开启开发人员模式，点击「创建应用」，填写 URL `https://mcp.dida365.com`，按提示完成 OAuth 授权。

#### TRAE

进入 **设置 > MCP > 添加 > 手动添加**，配置：

```json
{
  "mcpServers": {
    "dida365": {
      "url": "https://mcp.dida365.com"
    }
  }
}
```

保存后找到滴答清单，点击「前往验证」完成 OAuth 授权。

#### 其他支持 MCP 的客户端

通用配置要素：
- **传输协议**：Streamable HTTP（不支持 SSE）
- **URL**：`https://mcp.dida365.com`
- **认证**：OAuth 2.0（自动刷新 Token，无需重复登录）或 Bearer Token

### 连接验证

配置完成并授权后，在 AI 工具中执行以下验证：

> 「我今天有哪些任务？」

如果 MCP 连接正常，AI 会返回你滴答清单中的今日任务列表（空列表也说明连接成功）。如果返回错误或提示未连接，检查：
1. 是否已完成 OAuth 授权流程
2. 客户端是否支持 Streamable HTTP 协议
3. Bearer Token 是否过期或填写正确

### 常见问题

- **客户端不支持 Streamable HTTP？** 滴答清单 MCP 仅支持 Streamable HTTP，不支持 SSE。如果客户端只支持 SSE，暂时无法连接。
- **每次重启需要重新登录吗？** 不需要。OAuth 支持 Token 自动刷新，除非长时间未使用或主动撤销授权。
- **操作未按预期执行？** 尝试使用更明确的描述（指定清单名称、日期、优先级），或将复杂请求拆分为多步。

---

## 一、能力映射表

Agent 通过滴答 MCP 暴露的能力进行操作。每项能力对应一个 MCP 调用和一个本地降级路径：

| 用户需求 | MCP 调用 | 输入参数 | 返回数据 | 本地降级路径 |
|---------|---------|---------|---------|------------|
| 查看今日待办 | `get_today_tasks` | 无（按当前日期） | `[{id, title, due_date, priority, tags, project_id, completed_time}]` | 读取 `life-coach-data/planning/daily-plan.md` |
| 查看所有任务 | `get_all_tasks` | `project_id`（可选）、`filter`（可选） | `[{id, title, ...}]` | 读取 `life-coach-data/projects/projects.md` + `life-coach-data/planning/daily-plan.md` |
| 查看项目列表 | `get_projects` | 无 | `[{id, name, color, task_count}]` | 读取 `life-coach-data/projects/projects.md` |
| 创建任务 | `create_task` | `title`、`due_date`（可选）、`priority`（可选）、`tags`（可选）、`project_id`（可选） | `{id, title, ...}` | 写入本地 Markdown 并标记 `[待同步到滴答]` |
| 完成任务 | `complete_task` | `task_id` | `{success: true, completed_time}` | 本地 Markdown 标注 `[已完成-待同步]` |
| 取消完成任务 | `uncomplete_task` | `task_id` | `{success: true}` | 本地 Markdown 移除完成标记 |
| 删除任务 | `delete_task` | `task_id` | `{success: true}` | 本地 Markdown 标注 `[已删除-待同步]` |
| 更新任务 | `update_task` | `task_id`、更新字段（title/due_date/priority/tags/project_id） | `{id, title, ...}` | 更新本地 Markdown 内容 |
| 获取任务详情 | `get_task` | `task_id` | `{id, title, due_date, priority, tags, project_id, completed_time, content}` | 读取本地对应 Markdown 条目 |

### 调用频次限制

| 操作类型 | 建议频次 | 说明 |
|---------|---------|------|
| 读取（今日待办、项目列表） | 每次 CHECK 阶段最多 1 次 | 避免重复拉取；自动写入的幂等检查与此共用 |
| 读取（任务详情） | 按需调用 | 在 VERIFY 阶段查询确认 |
| 写入（创建/更新/删除） | 每次 ACT 阶段最多 1 次 | 遵循「一次一件事」原则 |
| 自动写入（Cron 填充） | 每节点最多 1 次 `create_task` | 09:30 / 14:30 / 20:00 填充后同步创建 |
| 自动写入（邮件意图） | 每封邮件最多 1 次 `create_task` | 邮件 `task_commitment` 即时创建 |
| 自动写入（幂等检查） | 每次处理最多 1 次 `get_today_tasks` | 与 CHECK 阶段共用读取，不重复拉取 |

## 二、自动写入规范

> 定义 Agent 在「承诺优先填充」「邮件意图 → 滴答」等自动场景下写入滴答的触发条件、幂等检查与字段推断规则。写入时遵循「不双写原则」：MCP 可用时滴答是任务状态的事实源，不在本地复制平行副本。

### 触发场景

| 触发场景 | 时机 | 说明 |
|---------|------|------|
| Cron 填充 | 各 Cron 节点填充计划后（09:30 / 14:30 / 20:00） | 将填充的今日计划项同步创建为滴答任务（`due_date: 今天`），晚间节点带 `tags`（晚间/精进/锻炼等） |
| 邮件意图 | Phase 4 分类为 `task_commitment` 时（Phase 5 Action G） | 用户邮件表达轻量任务承诺，即时创建为滴答任务并回复确认 |
| 用户确认 | 用户明确确认某项任务或计划调整后 | 计划对话中确认的任务，按 DECIDE → ACT 流程写入滴答，并在 VERIFY 阶段查询确认 |

### 幂等检查流程

创建任务前必须先查今日任务列表做幂等检查，避免重复创建：

```
get_today_tasks（获取今日已有待办）
    │
    ▼
标题模糊匹配（提取核心名词/动词 → 搜索关键词 → 相似度比较）
    │
    ├── 找到相似任务（相似度 > 70%）──→ 跳过创建，回复「这条已经记过了哦」
    │
    └── 未找到相似任务 ──→ create_task 创建新任务
            │
            ▼
        VERIFY（get_task 确认创建成功）
```

幂等检查要点：
1. 搜索关键词从用户原文的「核心名词 + 核心动词」中提取（如「交水电费」→ 搜索「水电费」）。
2. 标题比较允许措辞差异（模糊匹配），相似度 > 70% 视为同一任务，跳过创建。
3. 幂等检查的 `get_today_tasks` 与 CHECK 阶段共用一次读取，不重复拉取。

### 字段推断规则

从用户原文推断滴答任务字段。总规则：**时间词 → `due_date`，紧迫词 → `priority`，内容语义 → `tags`/`project_id`**。

| 推断目标 | 推断维度 | 匹配特征（关键词/句式） | 示例 | 推断结果 |
|---------|---------|----------------------|------|---------|
| `due_date` | 时间词 | `今天` + 动作动词 | 「今天要交水电费」 | 当日日期 |
| `due_date` | 时间词 | `明天` + 动作动词 | 「明天记得取快递」 | 次日日期 |
| `due_date` | 时间词 | `周末` + 动作动词 | 「周末大扫除」 | 本周六日期 |
| `due_date` | 时间词 | `下次` + 动作动词 | 「下次修一下门把手」 | 空（标记 `someday`，归入收集箱） |
| `due_date` | 时间词 | `得/要` + 动作动词 + `一下` | 「得给车做保养了」 | 空（标记 `this_week`，截止设本周日） |
| `priority` | 紧迫词 | `很急/必须/赶紧/立刻` | 「明天记得交报销，很急」 | 高 |
| `priority` | 紧迫词 | `不着急/有空/随便` | 「有空的时候看看那个课程」 | 低 |
| `tags` | 内容语义 | 看/读/学/课程/练习 | 「看《深度工作》」 | `精进` |
| `tags` | 内容语义 | 跑/健身/运动/游泳 | 「下次要去跑步」 | `锻炼` |
| `tags` | 内容语义 | 副业/项目/赛道/研究 | 「得研究一下这个副业方向」 | `副业` |

**推断规则：**

1. 用户原文包含明确日期（如「8 月 5 号交报告」）时，优先使用用户指定日期，不按模式推断。
2. 用户原文包含明确优先级信号（如「很急」「不着急」「必须」）时，优先使用用户表达，不按默认值。
3. 无明确紧迫词时，按时间词模式取默认优先级：今天 → 高、明天 → 中、周末 → 低、下次 → 低、得/要…一下 → 中。
4. `due_date` 为 `null` 且标记为 `someday` 的任务不设截止日期，归入「收集箱」或等价清单。
5. `due_date` 为 `null` 且标记为 `this_week` 的任务在滴答中设置截止日期为本周日。
6. Cron 填充写入的晚间任务默认带 `tags: [晚间, <类别>]`（类别为精进/休闲/锻炼/副业），便于次日 E1 统计覆盖快照。
7. 所有推断结果在写入时填入对应字段，并在回复用户时予以确认。

### 部署实测修正（2026-08 累积，与部署 prompt 对齐）

以下规则来自真实部署运行中暴露的问题，与 `engine/session-flow.md` Phase 4/5 及实际部署 prompt 保持一致：

**1. 实际 MCP 工具参数约定（三套风格，混用会静默失败或报 schema 错）**

| 工具 | 参数形式 | 示例 |
|------|---------|------|
| `create_task` | 全部包在 `task` 对象里 | `{ task: { title, content, projectId, dueDate, priority, ... } }` |
| `update_task` | 顶层 `task_id` + 嵌套 `task` 对象 | `{ task_id: "...", task: { id, projectId, title, ... } }` |
| `complete_task` | 顶层 snake_case 平铺 | `{ project_id: "...", task_id: "..." }` |

- `priority` 只接受枚举 `0 none / 1 low / 3 medium / 5 high`，其他值被静默降级为 0（不报错，只是优先级不对）。
- `dueDate` 用 ISO8601 带时区偏移（如 `2026-08-04T23:59:59+08:00`）；API 回显为 UTC（`+0000`）——时点相同，不要误读为失败。
- `update_task` 可能返回 `Expecting value: line 1 column 1`（服务端 JSON 解析错误，重试同样失败）→ 改用 `batch_update_tasks`（`tasks: [{ id, projectId, title, ... }]`），返回 `id2error` 为空即成功，之后用 `get_task_by_id` 验证。

**2. 幂等查重范围（防重复建 & 防漏建）**

- 查重范围必须覆盖同内容任务可能存在的**所有**项目：晚间块任务在四个类目项目里（见第 6 条映射），只查 💼工作任务 会漏判 → 重复创建。
- `search_task` 命中 ≠ 重复：先 `get_task_by_id` 查状态——`status=2`（已完成）且范围不同 → 不是重复，照常创建；只有 `status=0`（活跃）且同范围才跳过。标题相似度 <70% 不判重。
- **一句多动作**：一句话含 ≥2 个动宾结构（如「上传数据集，正在整理并压缩」）→ 每个动作**分别**查重、分别创建；主任务命中不得吞掉句中的新动作。「X 正在做」是对已有任务的进度描述（update_task），「新增任务：Y」是独立新任务（create_task）。

**3. 完成报告兜底**

用户报告完成但滴答搜不到对应任务（如「完成了一个新增任务 X」）→ **不得静默跳过**：说明 X 从未入滴答，先按任务承诺 `create_task` 补建（幂等查重），再按完成报告 `complete_task` 勾掉。

**4. meta-task 禁令**

「X 需要记录/同步到滴答清单」→ 任务标题是 **X**（实际工作本身），不是「同步 X 到滴答」这类记账任务——同步动作是隐含的，不是内容。误建了 meta-task 要先删再建正确的。

**5. 任务承诺识别扩展**

- 「在做/正在做/接下来做/最后一个时间块在做 X」这类**进行中任务**也归 `task_commitment`（不是状态更新）。
- 「新增任务：X」「完成了一个新增任务 X」是强触发器，即使含「完成」字样也优先按新增任务处理。
- 状态更新仅限非任务类陈述（「电影看完了」「cron 异常已解决」）；提到具体可执行任务（动宾结构）一律归任务承诺。

**6. 晚间块类别→项目映射（Cron 填充专用，项目名以用户实际清单为准）**

| 类别 | 项目 |
|------|------|
| 精进 | 📖精进 |
| 锻炼 | 🏃减肥指南 |
| 休闲 | 🎉娱乐 |
| 副业 | 🐙咸鱼 |

- 晚间填充任务**不写入 💼工作任务**（该项目的任务只来自用户邮件意图）；第三块留白跳过不创建（无内容的任务是噪音）。
- 每块创建前用 `list_undone_tasks_by_date` 查该类目项目今日任务 + `search_task` 搜标题，已有则引用不另建。

## 三、降级策略

### 分层降级

```
MCP 可用 ──→ 走 MCP
    │
    ▼
MCP 不可用 ──→ 走本地 Markdown + [待同步到滴答] 标记
    │
    ▼
MCP 恢复 ──→ 询问用户是否批量同步本地待同步条目
```

### 降级判定标准

| 条件 | 判定为 MCP 不可用 | 判定为 MCP 可用 |
|------|-----------------|----------------|
| MCP 调用返回错误 | 连续 2 次调用失败 | 1 次调用成功 |
| MCP 超时 | 单次调用超时（> 10 秒） | 正常响应 |
| MCP 未配置 | 工具列表中不存在滴答 MCP | 存在且可调用 |

### 降级输出格式

降级到本地 Markdown 时，在文件头或条目末尾添加标准标记：

```markdown
<!-- 滴答状态: 离线 @2026-07-29T14:30:00+08:00 -->
```

涉及具体任务条目时：

```markdown
- [ ] 完成项目报告 [待同步到滴答]
```

### 恢复同步流程

1. Agent 检测到 MCP 恢复可用后，不在无人请求时主动执行批量同步。
2. Agent 在下次合适的交互时机（如每日早安、计划调整）询问用户：「滴答清单已重新连接。本地有 N 条标记为 `待同步` 的条目，是否现在同步？」
3. 用户确认后，Agent 逐条读取本地待同步条目，通过 MCP 执行对应操作。
4. 同步完成后，移除本地条目的 `[待同步到滴答]` 标记，更新滴答状态标记为在线。
5. 同步失败的单条条目保持 `[待同步到滴答]` 标记，告知用户失败原因。

## 四、数据边界规则

### 职责边界

| 维度 | 滴答清单（Dida） | 本地 Markdown |
|------|----------------|--------------|
| 任务执行 | 管理任务状态、截止日期、优先级、标签 | 记录任务完成时的上下文和洞察 |
| 项目 | 维护项目层级和任务归属 | 记录项目愿景、里程碑、复盘 |
| 计划 | 反映最终确认的排期 | 记录计划过程中的权衡、备选方案 |
| 复盘 | 不涉及 | 完整的复盘记录、模式识别 |
| 长期愿景 | 不涉及 | 人生罗盘、长期目标、个人画像 |

### 滴答管什么

- 任务的**执行层面**：标题、截止时间、优先级、标签、项目归属、完成状态。
- 任务间的**依赖关系**：通过子任务和项目层级表达。
- 日常提醒和**重复任务**：通过滴答的提醒规则管理。

### 本地管什么

- **愿景与目标**：人生罗盘、三年愿景、年度目标，这些不进入滴答。
- **项目背景**：项目文档、里程碑规划、复盘记录、经验教训。
- **计划过程**：计划制定时的思考过程、权衡记录、备选方案（而非只有最终结果）。
- **用户画像**：精力模式、情绪信号、行为模式、沟通偏好。
- **复盘与洞察**：周/月/项目复盘、模式识别、策略有效性评估。

### 不跨边界原则

- Agent 不应将本地愿景数据（如 life-compass、价值排序）写入滴答标签或备注。
- Agent 不应将复盘洞察中的情绪模式写入滴答任务标题或优先级。
- Agent 不应利用滴答的项目层级来构建本地文档结构。

## 五、不双写原则

### 原则定义

**MCP 可用时，不在本地写任务副本**。滴答是任务状态的事实源（source of truth），本地文件不维护同一条任务的平行状态。

### 具体规则

1. **读取时**：MCP 可用 → 从滴答读取，不校验本地文件。MCP 不可用 → 从本地 Markdown 读取，标注 `[滴答离线]`。
2. **写入时**：MCP 可用 → 只写滴答，不同时写入本地 Markdown。MCP 不可用 → 只写本地 Markdown，标注 `[待同步到滴答]`。
3. **计划文件**：`life-coach-data/planning/daily-plan.md` 只记录「当日计划做什么」的意图和上下文，不复制滴答的任务状态。计划文件可以引用滴答任务（`滴答: "任务标题"`），但不维护该任务的完成进度。

### 例外

| 场景 | 处理方式 |
|------|---------|
| 用户明确要求同时保留本地记录 | 征得用户确认后，在 ACT 阶段同步写入本地和滴答 |
| 涉及 ISA 文档的任务 | ISA 文档记录完成标准，滴答记录执行状态，两套数据不相互覆盖 |
| 复盘时需要引用任务完成情况 | 从滴答查询后写入复盘文件，引用 `task_id` 和 `completed_time` |

### 同步冲突处理

如果本地 Markdown 和滴答清单的数据出现不一致（例如用户在滴答上手动修改了任务，而 Agent 在本地维护了旧版本）：

1. 以滴答的数据为事实源，覆盖本地标注。
2. 如果 Agent 检测到冲突，向用户展示差异并询问如何解决。
3. 不自动将本地数据推送给滴答（除非用户确认覆盖）。

## 六、滴答 MCP 操作对应的具体 Prompt 示例

以下示例展示 Agent 在 DECIDE → ACT → VERIFY 各阶段如何调用滴答 MCP。

### 示例 1：查看今日待办（CHECK 阶段）

```
你需要在 CHECK 阶段了解用户今日的待办事项。

调用滴答 MCP：
  - 工具：get_today_tasks

根据返回数据，提取：
  - 今日任务总数和已完成数。
  - 未完成任务中优先级最高的 3 项。
  - 是否有已过期未完成的任务。

返回结果整合为 Current State 摘要的「今日待办」部分。
不将滴答数据写入本地文件（遵循不双写原则）。
```

### 示例 2：创建任务（ACT 阶段，用户确认后）

```
用户在计划对话中确认了以下任务：
  - 标题："完成项目 Q3 报告"
  - 截止日期：2026-08-05
  - 优先级：p1（高）
  - 项目：工作

调用滴答 MCP：
  - 工具：create_task
  - 参数：
    title: "完成项目 Q3 报告"
    due_date: "2026-08-05"
    priority: 1
    project_id: <工作项目的 ID>
    tags: ["报告", "季度"]

调用后立即进入 VERIFY 阶段：
  - 调用 get_task，传入返回的 task_id。
  - 确认返回数据中的 completed_time 为空（任务尚未完成）。
  - 确认 title、due_date、priority 与创建参数一致。
```

### 示例 3：标记任务完成（ACT 阶段，用户确认后）

```
用户在回复中确认已完成「完成项目 Q3 报告」任务。

调用滴答 MCP：
  - 工具：complete_task
  - 参数：
    task_id: <任务的 ID>

调用后立即进入 VERIFY 阶段：
  - 调用 get_task，传入同一 task_id。
  - 确认返回数据中的 completed_time 不为空。
  - 记录 completed_time 的值到 LEARN 阶段的洞察记录。

验证通过后，在 LEARN 阶段写入 memory/daily/{today}.md：
  - 记录「用户完成了"完成项目 Q3 报告"」的事实。
  - 记录 completed_time 作为工具证据。
```

### 示例 4：改截止时间（ACT 阶段，用户确认后）

```
用户要求将「完成项目 Q3 报告」的截止时间从 8 月 5 日改为 8 月 10 日。

调用滴答 MCP：
  - 工具：update_task
  - 参数：
    task_id: <任务的 ID>
    due_date: "2026-08-10"

调用后立即进入 VERIFY 阶段：
  - 调用 get_task，传入同一 task_id。
  - 确认 due_date 已更新为 2026-08-10。
  - 确认其他字段未意外改动。

在 LEARN 阶段记录：
  - 截止时间变更事实（含变更前后的值）。
  - 如果这是同一任务第二次延期，将模式记录到 memory/long-term.md。

不将变更后的截止时间复制到本地计划文件（遵循不双写原则）。
但可以在计划文件中记录：「用户反馈 xx 任务延期至 8/10，原因是 xxx」。
```

### 示例 5：降级操作（MCP 不可用时的 ACT 阶段）

```
滴答 MCP 连续 2 次调用失败，判定为不可用。

将用户确认的任务信息写入本地 Markdown：

  文件：life-coach-data/planning/daily-plan.md
  条目：
  - [ ] 完成项目 Q3 报告 [待同步到滴答]（截止: 2026-08-05，优先级: 高）

  文件头部添加：
  <!-- 滴答状态: 离线 @2026-07-29T14:30:00+08:00 -->

在回复用户时附带说明：
  「滴答清单当前暂时不可用，任务已记录在本地计划文件中。待恢复后我会提醒你同步。」
```

## 七、AI 工具实现指导

### 7.1 MCP 调用生命周期

```
用户确认操作 → 构造 MCP 参数 → 调用 MCP → 解析返回值 → VERIFY → LEARN
```

- **构造参数**：从用户确认内容或当前上下文中提取必要字段。
- **调用**：一次调用一个操作，不批量调用。
- **解析**：检查返回值的 `success` 或 `errcode` 字段。
- **VERIFY**：通过查询 API 而非仅凭写入返回值确认（如 `complete_task` 后调用 `get_task` 查 `completed_time`）。
- **LEARN**：无论成功或失败，记录操作和结果到 memory。

### 7.2 错误处理

| 错误类型 | 处理方式 |
|---------|---------|
| MCP 返回 `errcode != 0` | 记录错误码和错误消息，告知用户操作失败及原因。不自动重试。 |
| MCP 超时 | 判定为不可用（参考降级策略），进入降级路径。 |
| MCP 返回数据与预期不符 | 不假定操作成功。通过独立的查询调用（`get_task`）确认实际状态。 |
| 参数校验失败 | 检查参数格式（如日期格式 YYYY-MM-DD、优先级范围 0-5），修正后重试一次。二次失败则告知用户。 |

### 7.3 安全约束

- 所有写入操作（创建、完成、删除、更新）**必须**在用户确认后执行。确认来自用户的明确自然语言回复，不来自 Agent 的推断。
- 删除操作为高风险操作，额外要求用户在确认中包含「删除」或等同关键词。
- 不通过滴答 MCP 读取用户的隐私数据（如带有明显个人密码信息的任务备注），如读取到应主动忽略。

### 7.4 在 algorithm 中的位置

| algorithm 步骤 | 滴答 MCP 使用场景 |
|---------------|-----------------|
| CHECK | `get_today_tasks`、`get_projects` 读当前状态 |
| GAP | 不直接使用（GAP 分析使用 CHECK 阶段已获取的数据） |
| DECIDE | 不直接使用（DECIDE 决定是否/如何操作滴答） |
| ACT | `create_task`、`complete_task`、`update_task`、`delete_task` |
| VERIFY | `get_task` 查询确认状态变更 |
| LEARN | 不直接使用（LEARN 将操作结果写入 memory） |

## References

- `system/algorithm.md`：6 步推理循环，定义滴答 MCP 在 CHECK、ACT、VERIFY 各阶段的调用位置。
- `coach/skills/planning/SKILL.md`：计划 skill 的滴答集成规则和降级策略。
- `integrations/tools.md`：通用工具适配层原则，滴答 MCP 作为具体实现遵循。
- `_reference/life-coach/templates/integrations/tools.md`：工具与数据来源模板。
