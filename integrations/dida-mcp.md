# 滴答清单 MCP 集成

> 定义 Agent 如何通过 MCP 协议与滴答清单（Dida）交互，包括能力映射、降级策略、数据边界和操作规范。

## 一、能力映射表

Agent 通过滴答 MCP 暴露的能力进行操作。每项能力对应一个 MCP 调用和一个本地降级路径：

| 用户需求 | MCP 调用 | 输入参数 | 返回数据 | 本地降级路径 |
|---------|---------|---------|---------|------------|
| 查看今日待办 | `get_today_tasks` | 无（按当前日期） | `[{id, title, due_date, priority, tags, project_id, completed_time}]` | 读取 `coach/data/daily-plan.md` |
| 查看所有任务 | `get_all_tasks` | `project_id`（可选）、`filter`（可选） | `[{id, title, ...}]` | 读取 `coach/data/projects.md` + `coach/data/daily-plan.md` |
| 查看项目列表 | `get_projects` | 无 | `[{id, name, color, task_count}]` | 读取 `coach/data/projects.md` |
| 创建任务 | `create_task` | `title`、`due_date`（可选）、`priority`（可选）、`tags`（可选）、`project_id`（可选） | `{id, title, ...}` | 写入本地 Markdown 并标记 `[待同步到滴答]` |
| 完成任务 | `complete_task` | `task_id` | `{success: true, completed_time}` | 本地 Markdown 标注 `[已完成-待同步]` |
| 取消完成任务 | `uncomplete_task` | `task_id` | `{success: true}` | 本地 Markdown 移除完成标记 |
| 删除任务 | `delete_task` | `task_id` | `{success: true}` | 本地 Markdown 标注 `[已删除-待同步]` |
| 更新任务 | `update_task` | `task_id`、更新字段（title/due_date/priority/tags/project_id） | `{id, title, ...}` | 更新本地 Markdown 内容 |
| 获取任务详情 | `get_task` | `task_id` | `{id, title, due_date, priority, tags, project_id, completed_time, content}` | 读取本地对应 Markdown 条目 |

### 调用频次限制

| 操作类型 | 建议频次 | 说明 |
|---------|---------|------|
| 读取（今日待办、项目列表） | 每次 CHECK 阶段最多 1 次 | 避免重复拉取 |
| 读取（任务详情） | 按需调用 | 在 VERIFY 阶段查询确认 |
| 写入（创建/更新/删除） | 每次 ACT 阶段最多 1 次 | 遵循「一次一件事」原则 |

## 二、降级策略

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

## 三、数据边界规则

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

## 四、不双写原则

### 原则定义

**MCP 可用时，不在本地写任务副本**。滴答是任务状态的事实源（source of truth），本地文件不维护同一条任务的平行状态。

### 具体规则

1. **读取时**：MCP 可用 → 从滴答读取，不校验本地文件。MCP 不可用 → 从本地 Markdown 读取，标注 `[滴答离线]`。
2. **写入时**：MCP 可用 → 只写滴答，不同时写入本地 Markdown。MCP 不可用 → 只写本地 Markdown，标注 `[待同步到滴答]`。
3. **计划文件**：`coach/data/daily-plan.md` 只记录「当日计划做什么」的意图和上下文，不复制滴答的任务状态。计划文件可以引用滴答任务（`滴答: "任务标题"`），但不维护该任务的完成进度。

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

## 五、滴答 MCP 操作对应的具体 Prompt 示例

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

  文件：coach/data/daily-plan.md
  条目：
  - [ ] 完成项目 Q3 报告 [待同步到滴答]（截止: 2026-08-05，优先级: 高）

  文件头部添加：
  <!-- 滴答状态: 离线 @2026-07-29T14:30:00+08:00 -->

在回复用户时附带说明：
  「滴答清单当前暂时不可用，任务已记录在本地计划文件中。待恢复后我会提醒你同步。」
```

## 六、AI 工具实现指导

### 6.1 MCP 调用生命周期

```
用户确认操作 → 构造 MCP 参数 → 调用 MCP → 解析返回值 → VERIFY → LEARN
```

- **构造参数**：从用户确认内容或当前上下文中提取必要字段。
- **调用**：一次调用一个操作，不批量调用。
- **解析**：检查返回值的 `success` 或 `errcode` 字段。
- **VERIFY**：通过查询 API 而非仅凭写入返回值确认（如 `complete_task` 后调用 `get_task` 查 `completed_time`）。
- **LEARN**：无论成功或失败，记录操作和结果到 memory。

### 6.2 错误处理

| 错误类型 | 处理方式 |
|---------|---------|
| MCP 返回 `errcode != 0` | 记录错误码和错误消息，告知用户操作失败及原因。不自动重试。 |
| MCP 超时 | 判定为不可用（参考降级策略），进入降级路径。 |
| MCP 返回数据与预期不符 | 不假定操作成功。通过独立的查询调用（`get_task`）确认实际状态。 |
| 参数校验失败 | 检查参数格式（如日期格式 YYYY-MM-DD、优先级范围 0-5），修正后重试一次。二次失败则告知用户。 |

### 6.3 安全约束

- 所有写入操作（创建、完成、删除、更新）**必须**在用户确认后执行。确认来自用户的明确自然语言回复，不来自 Agent 的推断。
- 删除操作为高风险操作，额外要求用户在确认中包含「删除」或等同关键词。
- 不通过滴答 MCP 读取用户的隐私数据（如带有明显个人密码信息的任务备注），如读取到应主动忽略。

### 6.4 在 algorithm 中的位置

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

## 附录 A：MCP 服务配置

> 本节为 AI 工具实现者提供滴答清单 MCP 服务的具体配置方法。

### A.1 前置条件

1. 拥有滴答清单账号（https://dida365.com 注册）
2. 滴答清单支持 MCP 协议的客户端版本（Web/桌面端均可）
3. 在滴答清单中生成 API Token（用于 MCP 认证）

### A.2 获取 API Token

1. 登录滴答清单网页版 https://dida365.com
2. 进入「设置」→「开发者」或「API」页面
3. 创建一个新的 API Token（或 Access Token）
4. 保存 Token 值（仅创建时可见一次）

### A.3 Claude Code 中的 MCP 配置

在项目根目录或用户目录的 `.mcp.json` 中添加滴答 MCP 服务：

```json
{
  "mcpServers": {
    "dida": {
      "command": "npx",
      "args": [
        "-y",
        "@dida365/mcp-server"
      ],
      "env": {
        "DIDA_API_TOKEN": "<你的 API Token>"
      }
    }
  }
}
```

或者使用环境变量方式（推荐，避免 Token 写入配置文件）：

```json
{
  "mcpServers": {
    "dida": {
      "command": "npx",
      "args": ["-y", "@dida365/mcp-server"]
    }
  }
}
```

然后在 shell 配置文件（`.bashrc`、`.zshrc`）或系统环境变量中设置：

```bash
export DIDA_API_TOKEN="<你的 API Token>"
```

### A.4 Cursor / Windsurf 中的 MCP 配置

在 `~/.cursor/mcp.json`（Cursor）或 IDE 的 MCP 设置中添加：

```json
{
  "mcpServers": {
    "dida": {
      "command": "npx",
      "args": ["-y", "@dida365/mcp-server"],
      "env": {
        "DIDA_API_TOKEN": "<你的 API Token>"
      }
    }
  }
}
```

### A.5 其他支持 MCP 的 AI 工具

对于任何支持 MCP 协议的工具，配置结构均类似：

| 字段 | 值 |
|------|-----|
| Server Name | `dida` |
| Command | `npx` |
| Args | `-y`, `@dida365/mcp-server` |
| 环境变量 | `DIDA_API_TOKEN=<你的 API Token>` |

### A.6 验证配置

配置完成后，验证 MCP 是否正常连接：

1. 重启 AI 工具（使 MCP 配置生效）
2. 在对话中要求 AI 工具调用滴答 MCP 的 `get_today_tasks` 工具
3. 如果返回任务列表（即使是空列表），说明连接成功
4. 如果返回错误，检查：
   - API Token 是否正确
   - 网络是否能访问 `dida365.com`
   - MCP server 包是否能通过 `npx` 下载

### A.7 配置失败时的处理

如果 MCP 配置不成功：

1. 检查 `npx @dida365/mcp-server` 是否能正常运行
2. 确认使用的滴答清单区域（中国区 `dida365.com` / 国际区 `ticktick.com`），Token 需在对应区域生成
3. 如仍无法配置，Agent 将使用本地 Markdown 降级模式，待办管理不会中断

### A.8 MCP Server 可用的 Tools 列表

配置成功后，滴答 MCP 通常暴露以下 tools：

| Tool 名称 | 功能 | 关键参数 |
|-----------|------|---------|
| `get_today_tasks` | 获取今日待办 | 无 |
| `get_all_tasks` | 获取全部任务 | `project_id`（可选）, `filter`（可选） |
| `get_projects` | 获取项目列表 | 无 |
| `create_task` | 创建任务 | `title`, `due_date`, `priority`, `tags`, `project_id` |
| `complete_task` | 完成任务 | `task_id` |
| `uncomplete_task` | 取消完成 | `task_id` |
| `delete_task` | 删除任务 | `task_id` |
| `update_task` | 更新任务 | `task_id`, 更新字段 |
| `get_task` | 获取任务详情 | `task_id` |

具体 tool 名称和参数以实际 MCP 返回的 tool schema 为准。AI 工具在启动时应读取 MCP 的 tool 列表，根据实际 schema 调用。
