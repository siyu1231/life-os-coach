# Life Coach Agent 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [x]`) syntax for tracking.

**Goal:** 构建 Life Coach Agent Skill Pack——一套工具无关的 Markdown 知识体系，融合 life-coach 教练能力和 LifeOS 系统化管理，支持 Cron + 腾讯企微邮件主动触达。

**Architecture:** 四层 Markdown 文件结构——engine 引擎层（cron/邮件/会话状态机）、system 系统层（Algorithm/ISA/Memory/TELOS/Verification）、coach 教练层（7步流程/ACT/7个skill）、integrations 集成层（滴答MCP/通用工具规则）。每层都是纯 Markdown 协议文档，由 AI 工具在运行时读取执行。

**Tech Stack:** 纯 Markdown，无代码依赖。仅定义规则、协议和模板。

## 全局约束

- 工具无关：所有文件不绑定特定 AI 工具（Claude Code/Codex/Cursor 等）
- Cron 由 AI 工具根据 `engine/cron-system.md` 自行配置
- 邮件使用腾讯企微 API `compose_send`，轮询收件箱接收回复
- 滴答清单可选接入，不可用时降级到本地 Markdown
- 所有 skill 内容源自 `_reference/life-coach/skills/` 对应文件
- 中文为主要语言，技术术语可用英文
- 用户数据存储在 `life-coach-data/` 目录，需写入 `.gitignore`

---

### Task 1: 项目脚手架——根目录文件 + .gitignore + 目录骨架

**Files:**
- Create: `.gitignore`（追加内容）
- Create: `README.md`
- Create: 空目录骨架（engine/, system/, coach/, coach/skills/, integrations/, templates/, examples/）

**Interfaces:**
- 无依赖（Task 1 是所有任务的基础）

- [ ] **Step 1: 创建所有空目录骨架**

```bash
mkdir -p engine system coach/skills integrations templates/profile templates/planning templates/projects templates/habits templates/reviews templates/memory examples
```

- [ ] **Step 2: 确保目录创建成功**

```bash
ls -d engine/ system/ coach/ coach/skills/ integrations/ templates/profile/ templates/planning/ templates/projects/ templates/habits/ templates/reviews/ templates/memory/ examples/
```

预期：所有目录存在，无错误输出。

- [ ] **Step 3: 更新 .gitignore——添加用户数据目录和 IDE 文件**

编辑 `.gitignore`，确保包含：

```gitignore
# 用户数据目录
life-coach-data/

# IDE
.idea/
.vscode/
*.swp
*.swo
*~

# OS
.DS_Store
Thumbs.db
```

验证：`cat .gitignore` 确认内容正确。

- [ ] **Step 4: 提交**

```bash
git add .gitignore && git add engine/ system/ coach/ integrations/ templates/ examples/ && git commit -m "feat: scaffold project directory structure and .gitignore"
```

---

### Task 2: Engine 引擎层——cron-system.md

**Files:**
- Create: `engine/cron-system.md`

**Interfaces:**
- 不依赖其他项目文件
- 引用外部 API：`https://timor.tech/api/holiday/info/{date}`（工作日判断）
- 产生的 Cron 规则被整个 engine 层使用

- [ ] **Step 1: 写入 cron-system.md**

内容覆盖：
1. 时间点定义表（9 个时间点：09:30 / 10:30 / 14:30 / 15:30 / 16:30 / 17:30 / 22:30 / 发送后+5min / 发送后+10min）
2. 每个时间点的运行条件（仅工作日 / 每日 / 动态）和触发动作
3. 工作日判断方法：调用 `timor.tech` API + 判断逻辑（`type.type === 0 && holiday === null` → 工作日）
4. 备份 API：`mxnzp.com`（需 app_id/app_secret）
5. 假期模式行为：日间跳过、晚间轻量复盘
6. 用户可参数化：时间点调整、启用/禁用、风格自定义
7. AI 工具配置 Cron 的指导说明

- [ ] **Step 2: 验证文件完整性**

```bash
grep -c "09:30\|10:30\|14:30\|15:30\|16:30\|17:30\|22:30" engine/cron-system.md
```
预期：至少匹配 7 个时间点。

```bash
grep -c "timor.tech\|mxnzp" engine/cron-system.md
```
预期：至少匹配 2。

```bash
grep -c "工作日\|休息日\|假期\|节假日" engine/cron-system.md
```
预期：至少匹配 4。

- [ ] **Step 3: 提交**

```bash
git add engine/cron-system.md && git commit -m "feat: add cron-system.md with workday-aware schedule rules"
```

---

### Task 3: Engine 引擎层——email-protocol.md

**Files:**
- Create: `engine/email-protocol.md`

**Interfaces:**
- 引用腾讯企微 API：`POST /cgi-bin/exmail/app/compose_send`（发送）、`POST /cgi-bin/exmail/app/get_email_alias`（查询邮箱）、获取收件箱列表 API、获取邮件内容 API
- 被 `engine/session-flow.md` 引用

- [ ] **Step 1: 写入 email-protocol.md**

内容覆盖：
1. 前置条件：企微应用配置、access_token 获取方式
2. 发送邮件 API：接口地址、请求参数（to/cc/bcc/subject/content/content_type）、HTML 正文模板结构
3. 查询应用邮箱 API：接口地址、返回示例
4. 接收邮件策略：轮询模式（不发回调服务，用 API 拉取）
5. 邮件正文模板：时间点卡片 + 上下文简述 + 具体问题 + 行动提示
6. 约束：正文不超过 200 字，每封一个核心问题，给选项优于开放式提问
7. AI 工具实现指导

- [ ] **Step 2: 验证文件完整性**

```bash
grep -c "compose_send\|get_email_alias" engine/email-protocol.md
```
预期：至少匹配 2。

```bash
grep -c "access_token\|ACCESS_TOKEN" engine/email-protocol.md
```
预期：至少匹配 1。

```bash
grep -c "轮询\|回调\|接收" engine/email-protocol.md
```
预期：至少匹配 2。

- [ ] **Step 3: 提交**

```bash
git add engine/email-protocol.md && git commit -m "feat: add email-protocol.md for WeCom email API"
```

---

### Task 4: Engine 引擎层——session-flow.md

**Files:**
- Create: `engine/session-flow.md`

**Interfaces:**
- 引用 `engine/cron-system.md` 的 Cron 规则
- 引用 `engine/email-protocol.md` 的邮件收发协议
- 引用 `system/algorithm.md` 的执行循环
- 引用 `system/memory-system.md` 的记忆写入规则
- 产生的会话状态机被所有 cron 触发流程使用

- [ ] **Step 1: 写入 session-flow.md**

内容覆盖：
1. 6 阶段状态机：采集 → 触达 → 接收 → 理解 → 行动 → 闭环
2. 每个阶段的触发条件、动作、出口
3. Phase 4（理解）的回复分类：纯状态 / 情绪困难 / 修改请求 / 深度问题
4. 回复轮询：发送后 5min 和 10min 各一次
5. 一次会话只处理第一封有效回复
6. 边界情况：无回复、邮件发送失败、节假日跳过
7. AI 工具实现状态机的指导

- [ ] **Step 2: 验证文件完整性**

```bash
grep -c "Phase\|采集\|触达\|接收\|理解\|行动\|闭环" engine/session-flow.md
```
预期：至少匹配 6。

```bash
grep -c "5min\|10min\|轮询" engine/session-flow.md
```
预期：至少匹配 2。

- [ ] **Step 3: 提交**

```bash
git add engine/session-flow.md && git commit -m "feat: add session-flow.md with 6-phase email conversation state machine"
```

---

### Task 5: System 系统层——algorithm.md + isa-system.md

**Files:**
- Create: `system/algorithm.md`
- Create: `system/isa-system.md`

**Interfaces:**
- `algorithm.md`：引用 `system/memory-system.md` 和 `integrations/dida-mcp.md`
- `isa-system.md`：被 `algorithm.md` 的 PLAN 阶段引用
- 两者无依赖关系，可同任务创建

- [ ] **Step 1: 写入 algorithm.md**

内容覆盖：
1. 6 步循环：CHECK → GAP → DECIDE → ACT → VERIFY → LEARN
2. 每一步的详细说明：做什么、读什么、输出什么
3. CHECK：读记忆 + 滴答/本地计划 → 了解 Current State
4. GAP：对比计划 vs 实际 → 发现差距和状态信号
5. DECIDE：根据差距类型 → 决定动作（状态同步 / coach 介入 / 计划调整 / 复盘触发）
6. ACT：执行（发邮件 / 写文件 / 调用 coach skill）
7. VERIFY：工具确认（写入验证、邮件发送状态确认）
8. LEARN：记录洞察到 memory
9. 核心原则：派生自 LifeOS——"每个任务都是 Current→Ideal 的爬山"、"没有证据不能说做完了"

- [ ] **Step 2: 写入 isa-system.md**

内容覆盖：
1. ISA 定义：Ideal State Artifact——「做完」写成可验证的声明
2. ISA 格式模板：Goal / Vision / Out of Scope / Claims / Anti-claims
3. 何时需要 ISA：任何大于「看一眼就能回答」的任务
4. 何时不需要 ISA：纯状态更新、简单问答
5. Coach 场景的 ISA 使用方式：不需要代码级严格性，但保留核心精神
6. 示例 ISA 文档
7. 与滴答清单的关系：滴答任务可映射为 ISA Claims

- [ ] **Step 3: 验证 algorithm.md**

```bash
grep -c "CHECK\|GAP\|DECIDE\|ACT\|VERIFY\|LEARN" system/algorithm.md
```
预期：至少匹配 6。

- [ ] **Step 4: 验证 isa-system.md**

```bash
grep -c "Goal\|Vision\|Out of Scope\|Claims\|Anti-claims" system/isa-system.md
```
预期：至少匹配 4。

- [ ] **Step 5: 提交**

```bash
git add system/algorithm.md system/isa-system.md && git commit -m "feat: add algorithm.md and isa-system.md for LifeOS-style execution loop and ISA"
```

---

### Task 6: System 系统层——memory-system.md + verification.md

**Files:**
- Create: `system/memory-system.md`
- Create: `system/verification.md`

**Interfaces:**
- `memory-system.md`：被 `system/algorithm.md` 的 CHECK/LEARN 阶段引用，被 `engine/session-flow.md` 的 Phase 5 引用
- `verification.md`：被 `system/algorithm.md` 的 VERIFY 阶段引用

- [ ] **Step 1: 写入 memory-system.md**

完整迁移 `_reference/life-coach/references/memory-system.md` 的内容，并做以下改造：
1. 继承「先少后多、单一事实源、先记录再提炼」原则
2. 继承「写入前确认」和「事实/假设/洞察分开」规则
3. 继承目录结构和文件职责说明
4. **新增**：四级写入安全等级（Tier A-D），融合 LifeOS 的 Tier 概念
5. **新增**：与滴答清单的数据边界（滴答管任务执行，本地管愿景/项目/复盘）
6. **新增**：拆分阈值规则（满足 2 条才拆分新文件）

- [ ] **Step 2: 写入 verification.md**

内容覆盖：
1. 三级验证：数据验证（数字/文件/时间戳）、证据验证（具体事实）、觉察验证（用户表达）
2. 原则："数据优先，感觉补充"
3. 滴答场景的验证示例（完成率来自滴答数据）
4. 邮件场景的验证示例（邮件回复作为证据）
5. Coach 场景的验证示例（用户原话引用）
6. "以为做完了"的常见陷阱（不检查文件、不确认邮件发送、凭感觉判断）

- [ ] **Step 3: 验证 memory-system.md**

```bash
grep -c "Tier\|安全等级\|写入前确认\|直接写入\|永远不写" system/memory-system.md
```
预期：至少匹配 3。

```bash
grep -c "profile\|planning\|projects\|reviews\|habits\|memory" system/memory-system.md
```
预期：至少匹配 5。

- [ ] **Step 4: 验证 verification.md**

```bash
grep -c "数据验证\|证据验证\|觉察验证" system/verification.md
```
预期：至少匹配 3。

- [ ] **Step 5: 提交**

```bash
git add system/memory-system.md system/verification.md && git commit -m "feat: add memory-system.md (4-tier safety writes) and verification.md (3-level verification)"
```

---

### Task 7: System 系统层——telos-system.md

**Files:**
- Create: `system/telos-system.md`

**Interfaces:**
- 引用 `templates/profile/life-compass.md` 模板
- 被 `coach/skills/life-vision/` 的流程引用

- [ ] **Step 1: 写入 telos-system.md**

内容覆盖：
1. TELOS 概念：Current State → Ideal State 的方向系统（融合 LifeOS TELOS 和 life-coach 价值观罗盘）
2. 人生 8+1 领域：工作/学习/健康/关系/财务/精神/休闲/环境 + 1 个自定义
3. 每个领域的结构：当前状态 / 理想状态 / 阶段主题 / 不做什么
4. 如何更新 TELOS：每月/每季度 + 重大变化时
5. ISA 与 TELOS 的关系：ISA 是 TELOS 领域的具体化
6. 与 life-vision skill 的分工：life-vision 负责对话澄清，TELOS 负责结构化存储

- [ ] **Step 2: 验证文件**

```bash
grep -c "TELOS\|当前状态\|理想状态\|阶段主题\|8+1\|领域" system/telos-system.md
```
预期：至少匹配 4。

- [ ] **Step 3: 提交**

```bash
git add system/telos-system.md && git commit -m "feat: add telos-system.md unifying vision/goals/values life model"
```

---

### Task 8: Coach 教练层——coaching-process.md + act-core.md

**Files:**
- Create: `coach/coaching-process.md`
- Create: `coach/act-core.md`

**Interfaces:**
- 源文件：`_reference/life-coach/references/coaching-process.md` 和 `_reference/life-coach/references/act-core.md`
- 被所有 7 个 coach skills 引用

- [ ] **Step 1: 写入 coaching-process.md**

完整迁移 `_reference/life-coach/references/coaching-process.md` 的全部内容，并在开头增加：
1. **邮件场景适配说明**：每封一个核心问题、200 字上限、选项优于开放式提问
2. **与 CLI/Chat 分工**：邮件做触达和状态采集，CLI/Chat 做深度教练
3. **何时引导到 CLI/Chat**：连续 2+ 次出现深度问题、用户表达强烈情绪、需要多轮澄清

保持原有的 7 步流程、微技术清单、强情绪场景、边界与转介规则不变。

- [ ] **Step 2: 写入 act-core.md**

完整迁移 `_reference/life-coach/references/act-core.md` 的全部内容，并在末尾增加：
1. **邮件场景微干预速查表**：4 种高频回复（状态不好/不想做/我不行/明天再说）各配 1-2 句 ACT 回应
2. **ACT 邮件对话约束**：一次只用一个技术、不要堆砌术语、用行动选项收尾

保持原有的六大过程、话术速查表、转交信号不变。

- [ ] **Step 3: 验证 coaching-process.md**

对比源文件行数（约 249 行），确保不少于源文件行数：
```bash
wc -l coach/coaching-process.md
```

```bash
grep -c "建立安全\|接住情绪\|澄清主题\|促进觉察\|聚焦选择\|共创行动\|整合与收尾" coach/coaching-process.md
```
预期：至少匹配 7。

- [ ] **Step 4: 验证 act-core.md**

对比源文件行数（约 248 行），确保不少于源文件行数：
```bash
wc -l coach/act-core.md
```

```bash
grep -c "认知解离\|接纳\|当下觉察\|以己为景\|价值观澄清\|承诺行动\|选择点" coach/act-core.md
```
预期：至少匹配 7。

- [ ] **Step 5: 提交**

```bash
git add coach/coaching-process.md coach/act-core.md && git commit -m "feat: add coaching-process.md and act-core.md from life-coach, with email-mode adaptations"
```

---

### Task 9: Coach Skills——life-vision

**Files:**
- Create: `coach/skills/life-vision/SKILL.md`
- Create: `coach/skills/life-vision/mail-trigger.md`
- Create: `coach/skills/life-vision/references/workflow.md`
- Create: `coach/skills/life-vision/references/value-compass.md`
- Create: `coach/skills/life-vision/references/designing-your-life.md`
- Create: `coach/skills/life-vision/references/life-design-toolkit.md`
- Create: `coach/skills/life-vision/references/cases.md`

**Interfaces:**
- 源文件：`_reference/life-coach/skills/life-vision/` 下所有文件
- `coach/coaching-process.md`、`coach/act-core.md`、`system/telos-system.md`

- [ ] **Step 1: 创建目录**

```bash
mkdir -p coach/skills/life-vision/references
```

- [ ] **Step 2: 写入 SKILL.md**

以 `_reference/life-coach/skills/life-vision/SKILL.md` 为基础，新增 frontmatter `mail-trigger` 字段：迷茫信号、方向感缺失、多次计划偏离且非执行问题 → 引导到 CLI/Chat。

- [ ] **Step 3: 写入 mail-trigger.md**

内容：邮件触发信号定义 + 邮件环境下的轻量回应模板 + 何时引导用户开 Chat 深度对话。

- [ ] **Step 4: 迁移 4 个 references 文件**

从 `_reference/life-coach/skills/life-vision/references/` 复制 `workflow.md`、`value-compass.md`、`designing-your-life.md`、`life-design-toolkit.md`、`cases.md`。

- [ ] **Step 5: 验证所有文件非空**

```bash
for f in coach/skills/life-vision/SKILL.md coach/skills/life-vision/mail-trigger.md coach/skills/life-vision/references/*.md; do echo "$f: $(wc -l < "$f") lines"; done
```
预期：每个文件行数 > 0。

- [ ] **Step 6: 提交**

```bash
git add coach/skills/life-vision/ && git commit -m "feat: add life-vision coach skill with email triggers"
```

---

### Task 10: Coach Skills——planning

**Files:**
- Create: `coach/skills/planning/SKILL.md`
- Create: `coach/skills/planning/mail-trigger.md`
- Create: `coach/skills/planning/references/workflow.md`
- Create: `coach/skills/planning/references/planning-toolkit.md`
- Create: `coach/skills/planning/references/cognitive-concepts.md`
- Create: `coach/skills/planning/references/cases.md`

**Interfaces:**
- 源文件：`_reference/life-coach/skills/planning/`
- `system/algorithm.md`、`system/isa-system.md`、`integrations/dida-mcp.md`

- [ ] **Step 1: 创建目录**

```bash
mkdir -p coach/skills/planning/references
```

- [ ] **Step 2: 写入 SKILL.md**

以源文件为基础，**新增滴答集成引用**：planning skill 优先走滴答 MCP，降级走本地 Markdown。
新增 frontmatter `mail-trigger` 字段。

- [ ] **Step 3: 写入 mail-trigger.md**

内容：每日启动信号 + 用户请求改计划信号 + 收尾未完成项处置信号。每种信号对应的邮件模板。

- [ ] **Step 4: 迁移 4 个 references 文件**

从 `_reference/life-coach/skills/planning/references/` 复制全部文件。

- [ ] **Step 5: 验证并提交**

```bash
for f in coach/skills/planning/SKILL.md coach/skills/planning/mail-trigger.md coach/skills/planning/references/*.md; do echo "$f: $(wc -l < "$f") lines"; done
```
预期：每个文件行数 > 0。

```bash
git add coach/skills/planning/ && git commit -m "feat: add planning coach skill with Dida MCP integration"
```

---

### Task 11: Coach Skills——procrastination-execution

**Files:**
- Create: `coach/skills/procrastination-execution/SKILL.md`
- Create: `coach/skills/procrastination-execution/mail-trigger.md`
- Create: `coach/skills/procrastination-execution/references/workflow.md`
- Create: `coach/skills/procrastination-execution/references/procrastination-toolkit.md`
- Create: `coach/skills/procrastination-execution/references/cognitive-concepts.md`
- Create: `coach/skills/procrastination-execution/references/cases.md`

**Interfaces:**
- 源文件：`_reference/life-coach/skills/procrastination-execution/`
- `coach/act-core.md`（ACT 技术应用于拖延场景）

- [ ] **Step 1: 创建目录并写入 SKILL.md + mail-trigger.md**

mail-trigger 信号：连续 2 个时段同一任务未推进 / "不想做" / "没动力" / "等一下"。触发动作：压缩任务大小、2 分钟启动法、换环境建议。

- [ ] **Step 2: 迁移 4 个 references 文件并提交**

```bash
mkdir -p coach/skills/procrastination-execution/references
cp _reference/life-coach/skills/procrastination-execution/references/*.md coach/skills/procrastination-execution/references/
git add coach/skills/procrastination-execution/ && git commit -m "feat: add procrastination-execution coach skill"
```

---

### Task 12: Coach Skills——review

**Files:**
- Create: `coach/skills/review/SKILL.md`
- Create: `coach/skills/review/mail-trigger.md`
- Create: `coach/skills/review/references/workflow.md`
- Create: `coach/skills/review/references/review-toolkit.md`
- Create: `coach/skills/review/references/cognitive-concepts.md`
- Create: `coach/skills/review/references/cases.md`

**Interfaces:**
- 源文件：`_reference/life-coach/skills/review/`
- `system/verification.md`（复盘中的验证）

- [ ] **Step 1: 创建目录并写入 SKILL.md + mail-trigger.md**

mail-trigger 信号：22:30 复盘时间触发 / 用户说"这周又废了"。晚间复盘模板：3-5 个结构化问题，工作日和假期版本。日间收尾（17:30）的复盘问题。

- [ ] **Step 2: 迁移 4 个 references 文件并提交**

```bash
mkdir -p coach/skills/review/references
# Copy reference files from source
git add coach/skills/review/ && git commit -m "feat: add review coach skill with evening and daily wrap-up templates"
```

---

### Task 13: Coach Skills——habit-design + energy-management + complex-problem-solving

**Files:**
- Create: `coach/skills/habit-design/` (SKILL.md + mail-trigger.md + references/)
- Create: `coach/skills/energy-management/` (SKILL.md + mail-trigger.md + references/)
- Create: `coach/skills/complex-problem-solving/` (SKILL.md + mail-trigger.md + references/)

**Interfaces:**
- 源文件：`_reference/life-coach/skills/habit-design/`、`energy-management/`、`complex-problem-solving/`
- 跨 Skill 转交规则（继承 life-coach AGENTS.md）

- [ ] **Step 1: habit-design**

```bash
mkdir -p coach/skills/habit-design/references
cp _reference/life-coach/skills/habit-design/references/*.md coach/skills/habit-design/references/
```

mail-trigger 信号：同一偏离在复盘中连续 3+ 次出现。

- [ ] **Step 2: energy-management**

```bash
mkdir -p coach/skills/energy-management/references
cp _reference/life-coach/skills/energy-management/references/*.md coach/skills/energy-management/references/
```

mail-trigger 信号："累"/"困"/"没精力"/连续 3+ 时段低完成率。

- [ ] **Step 3: complex-problem-solving**

```bash
mkdir -p coach/skills/complex-problem-solving/references
cp _reference/life-coach/skills/complex-problem-solving/references/*.md coach/skills/complex-problem-solving/references/
```

mail-trigger 信号：用户描述多因素纠缠且无法清晰表达「问题是什么」→ 引导到 CLI/Chat。

- [ ] **Step 4: 写入三个 SKILL.md 和 mail-trigger.md**

每个 skill 写入 SKILL.md（基于源文件 + 增加 mail-trigger frontmatter）+ 新建 mail-trigger.md。

- [ ] **Step 5: 验证并提交**

```bash
for skill in habit-design energy-management complex-problem-solving; do
  echo "=== $skill ==="
  for f in coach/skills/$skill/SKILL.md coach/skills/$skill/mail-trigger.md; do
    echo "$f: $(wc -l < "$f") lines"
  done
done
```
预期：每个 SKILL.md > 0 行，每个 mail-trigger.md > 0 行。

```bash
git add coach/skills/habit-design/ coach/skills/energy-management/ coach/skills/complex-problem-solving/ && git commit -m "feat: add habit-design, energy-management, and complex-problem-solving coach skills"
```

---

### Task 14: Integrations 集成适配层

**Files:**
- Create: `integrations/dida-mcp.md`
- Create: `integrations/tools.md`

**Interfaces:**
- 被 `system/algorithm.md`、`coach/skills/planning/SKILL.md` 引用

- [ ] **Step 1: 写入 dida-mcp.md**

内容覆盖：
1. 能力映射表：读今日任务 / 读项目列表 / 创建任务 / 更新任务状态 / 改截止时间 → 对应的 MCP 调用和本地降级路径
2. 降级策略：MCP 可用 → 走 MCP；不可用 → 走本地 Markdown + `[待同步到滴答]` 标记；MCP 恢复 → 询问批量同步
3. 数据边界规则：滴答管任务执行，本地管愿景/项目/复盘
4. 不双写原则：MCP 可用时不写 Markdown 任务
5. 滴答 MCP 操作对应的具体 prompt 示例
6. AI 工具实现指导

- [ ] **Step 2: 写入 tools.md**

内容覆盖：
1. 接入原则：读取更主动，写入更谨慎
2. 降级不中断：外部工具不可用 → 降级到本地 Markdown
3. 写入前确认：创建/修改/删除外部数据需用户确认
4. 新工具扩展流程：用户提供 → 记录到此文件 → AI 读取生效
5. 未来可扩展工具类型：日历 MCP / Notion API / Obsidian 等

- [ ] **Step 3: 验证并提交**

```bash
grep -c "MCP\|降级\|滴答\|dida" integrations/dida-mcp.md
```
预期：至少匹配 3。

```bash
git add integrations/ && git commit -m "feat: add Dida MCP integration rules and general tool adapter"
```

---

### Task 15: Templates 用户数据模板

**Files:**
- Create: `templates/profile/user.md`
- Create: `templates/profile/life-compass.md`
- Create: `templates/profile/energy-profile.md`（可选拆分模板）
- Create: `templates/planning/daily-plan.md`
- Create: `templates/planning/weekly-plan.md`
- Create: `templates/projects/projects.md`
- Create: `templates/habits/habit-tracker.md`
- Create: `templates/reviews/review-log.md`
- Create: `templates/memory/daily.md`
- Create: `templates/memory/long-term.md`
- Create: `templates/integrations/tools.md`

**Interfaces:**
- 源文件：`_reference/life-coach/templates/` 下所有文件
- 被 `system/memory-system.md` 引用
- 安装时被 AI 工具复制到用户数据目录

- [ ] **Step 1: 复制 life-coach 模板**

```bash
find _reference/life-coach/templates -type f -name "*.md" | while read f; do
  target="${f#_reference/life-coach/}"
  mkdir -p "$(dirname "$target")"
  cp "$f" "$target"
done
```

- [ ] **Step 2: 新增模板文件**

创建 `templates/integrations/tools.md`：外部工具配置模板（MCP 名称、能力、降级路径）。

创建 `templates/memory/long-term.md`：跨文件洞察索引模板。

- [ ] **Step 3: 验证所有模板存在**

```bash
for f in templates/profile/user.md templates/profile/life-compass.md templates/planning/daily-plan.md templates/planning/weekly-plan.md templates/projects/projects.md templates/habits/habit-tracker.md templates/reviews/review-log.md templates/memory/daily.md templates/memory/long-term.md templates/integrations/tools.md; do
  echo "$f: $(wc -l < "$f") lines"
done
```
预期：每个文件 > 0 行。

- [ ] **Step 4: 提交**

```bash
git add templates/ && git commit -m "feat: add user data templates from life-coach with additional files"
```

---

### Task 16: Examples + README

**Files:**
- Create: `examples/morning-checkin.example.md`（晨间启动完整邮件示例）
- Create: `examples/procrastination-email.example.md`（拖延应对邮件示例）
- Create: `examples/evening-review.example.md`（晚间复盘邮件示例）
- Create: `examples/cli-deep-coaching.example.md`（CLI/Chat 深度教练示例）
- Create: `README.md`

**Interfaces:**
- 引用 `_reference/life-coach/README.md` 的理论来源声明
- 向用户展示完整的使用场景

- [ ] **Step 1: 写入 4 个示例文件**

每个示例展示完整的交互回合：cron 触发 → 邮件发送 → 用户回复 → Agent 处理 → 确认邮件 → 文件变更前后对比。

- [ ] **Step 2: 写入 README.md**

内容覆盖：
1. 项目定位 + 一句话描述
2. 核心能力表
3. 架构图（ASCII art）
4. 快速开始（给 AI 工具一句话）
5. 安装流程
6. 理论来源和致谢（life-coach 和 LifeOS）
7. 安全边界
8. 隐私说明

- [ ] **Step 3: 验证并提交**

```bash
for f in examples/*.md; do echo "$f: $(wc -l < "$f") lines"; done
```
预期：每个 > 20 行。

```bash
wc -l README.md
```
预期：> 50 行。

```bash
git add examples/ README.md && git commit -m "feat: add usage examples and project README"
```

---

### Task 17: AGENTS.md——总调度人格

**Files:**
- Create: `AGENTS.md`

**Interfaces:**
- 项目的入口文件，被 AI 工具在启动时读取
- 路由表引用所有 layer 文件

- [ ] **Step 1: 写入 AGENTS.md**

以 `_reference/life-coach/AGENTS.md` 为基础，做以下改造：

1. **角色**：保留 life-coach 的「温暖、清晰、务实的人生项目教练」人格
2. **路由表**：扩展为四层路由——engine 规则 / system 循环 / coach skill / integrations 规则
3. **新增引擎感知**：AI 工具应首先理解自己需要做什么——读 cron-system.md 配置 cron、读 email-protocol.md 配置邮件
4. **新增安装流程入口**：当用户第一次给 AI 这个项目时，AI 应该检测是否已安装（cron 是否配置、邮件是否可用、用户数据是否创建），若未安装则走安装向导
5. **共享沟通风格**：完整保留
6. **安全边界**：完整保留 + 新增邮件隐私规则（不记录邮件内容中与教练无关的私人信息）
7. **新增输出模式**：邮件输出 vs CLI/Chat 输出
8. **跨 Skill 转交**：完整保留

- [ ] **Step 2: 验证 AGENTS.md**

```bash
grep -c "路由\|skill\|engine\|system\|coach\|integration" AGENTS.md
```
预期：至少匹配 5。

```bash
grep -c "安装\|cron\|邮件\|email" AGENTS.md
```
预期：至少匹配 2。

```bash
grep -c "安全\|边界\|转介" AGENTS.md
```
预期：至少匹配 2。

- [ ] **Step 3: 提交**

```bash
git add AGENTS.md && git commit -m "feat: add AGENTS.md as master scheduler with install wizard and 4-layer routing"
```

---

### Task 18: 最终验证——文件树完整性检查

**Files:**
- 无新建文件（只读验证）

- [ ] **Step 1: 验证完整文件树**

```bash
echo "=== Expected files ==="
cat <<'EOF'
AGENTS.md
README.md
.gitignore
engine/cron-system.md
engine/email-protocol.md
engine/session-flow.md
system/algorithm.md
system/isa-system.md
system/memory-system.md
system/telos-system.md
system/verification.md
coach/coaching-process.md
coach/act-core.md
coach/skills/life-vision/SKILL.md
coach/skills/life-vision/mail-trigger.md
coach/skills/life-vision/references/workflow.md
coach/skills/life-vision/references/value-compass.md
coach/skills/life-vision/references/designing-your-life.md
coach/skills/life-vision/references/life-design-toolkit.md
coach/skills/life-vision/references/cases.md
coach/skills/planning/SKILL.md
coach/skills/planning/mail-trigger.md
coach/skills/planning/references/workflow.md
coach/skills/planning/references/planning-toolkit.md
coach/skills/planning/references/cognitive-concepts.md
coach/skills/planning/references/cases.md
coach/skills/procrastination-execution/SKILL.md
coach/skills/procrastination-execution/mail-trigger.md
coach/skills/procrastination-execution/references/workflow.md
coach/skills/procrastination-execution/references/procrastination-toolkit.md
coach/skills/procrastination-execution/references/cognitive-concepts.md
coach/skills/procrastination-execution/references/cases.md
coach/skills/review/SKILL.md
coach/skills/review/mail-trigger.md
coach/skills/review/references/workflow.md
coach/skills/review/references/review-toolkit.md
coach/skills/review/references/cognitive-concepts.md
coach/skills/review/references/cases.md
coach/skills/habit-design/SKILL.md
coach/skills/habit-design/mail-trigger.md
coach/skills/energy-management/SKILL.md
coach/skills/energy-management/mail-trigger.md
coach/skills/complex-problem-solving/SKILL.md
coach/skills/complex-problem-solving/mail-trigger.md
integrations/dida-mcp.md
integrations/tools.md
templates/profile/user.md
templates/profile/life-compass.md
templates/planning/daily-plan.md
templates/planning/weekly-plan.md
templates/projects/projects.md
templates/habits/habit-tracker.md
templates/reviews/review-log.md
templates/memory/daily.md
templates/memory/long-term.md
templates/integrations/tools.md
examples/morning-checkin.example.md
examples/procrastination-email.example.md
examples/evening-review.example.md
examples/cli-deep-coaching.example.md
EOF
echo "=== Checking file existence ==="
while IFS= read -r path; do
  [ -z "$path" ] && continue
  if [ -f "$path" ]; then
    echo "✅ $path"
  else
    echo "❌ MISSING: $path"
  fi
done < <(cat <<'EOF2'
AGENTS.md
README.md
.gitignore
engine/cron-system.md
engine/email-protocol.md
engine/session-flow.md
system/algorithm.md
system/isa-system.md
system/memory-system.md
system/telos-system.md
system/verification.md
coach/coaching-process.md
coach/act-core.md
coach/skills/life-vision/SKILL.md
coach/skills/life-vision/mail-trigger.md
coach/skills/life-vision/references/workflow.md
coach/skills/life-vision/references/value-compass.md
coach/skills/life-vision/references/designing-your-life.md
coach/skills/life-vision/references/life-design-toolkit.md
coach/skills/life-vision/references/cases.md
coach/skills/planning/SKILL.md
coach/skills/planning/mail-trigger.md
coach/skills/planning/references/workflow.md
coach/skills/planning/references/planning-toolkit.md
coach/skills/planning/references/cognitive-concepts.md
coach/skills/planning/references/cases.md
coach/skills/procrastination-execution/SKILL.md
coach/skills/procrastination-execution/mail-trigger.md
coach/skills/procrastination-execution/references/workflow.md
coach/skills/procrastination-execution/references/procrastination-toolkit.md
coach/skills/procrastination-execution/references/cognitive-concepts.md
coach/skills/procrastination-execution/references/cases.md
coach/skills/review/SKILL.md
coach/skills/review/mail-trigger.md
coach/skills/review/references/workflow.md
coach/skills/review/references/review-toolkit.md
coach/skills/review/references/cognitive-concepts.md
coach/skills/review/references/cases.md
coach/skills/habit-design/SKILL.md
coach/skills/habit-design/mail-trigger.md
coach/skills/energy-management/SKILL.md
coach/skills/energy-management/mail-trigger.md
coach/skills/complex-problem-solving/SKILL.md
coach/skills/complex-problem-solving/mail-trigger.md
integrations/dida-mcp.md
integrations/tools.md
templates/profile/user.md
templates/profile/life-compass.md
templates/planning/daily-plan.md
templates/planning/weekly-plan.md
templates/projects/projects.md
templates/habits/habit-tracker.md
templates/reviews/review-log.md
templates/memory/daily.md
templates/memory/long-term.md
templates/integrations/tools.md
examples/morning-checkin.example.md
examples/procrastination-email.example.md
examples/evening-review.example.md
examples/cli-deep-coaching.example.md
EOF2
)
```

预期：所有文件显示 ✅。

- [ ] **Step 2: 验证 reference 文件完整复制**

```bash
# 验证 habit-design/energy-management/complex-problem-solving 的 references 目录
for skill in habit-design energy-management complex-problem-solving; do
  echo "=== $skill references ==="
  ls coach/skills/$skill/references/
done
```
预期：每个 skill 的 references/ 目录包含来自 `_reference/life-coach/skills/<skill>/references/` 的所有文件。

- [ ] **Step 3: 最终提交（如有缺失文件则补齐后再提交）**

```bash
git add -A && git status
```

---

## 自审

**1. Spec 覆盖检查：**

| Spec 章节 | 覆盖任务 |
|-----------|---------|
| 3. Engine 引擎层 (cron/email/session) | Task 2-4 |
| 4. System 系统层 (algorithm/ISA/memory/TELOS/verify) | Task 5-7 |
| 5. Coach 教练层 (coaching/ACT/7 skills) | Task 8-13 |
| 6. Integrations 集成层 (dida/tools) | Task 14 |
| 7. 交互模式 (邮件 vs CLI/Chat) | Task 8 (coaching-process.md), Task 16 (examples) |
| 8. 首次安装流程 | Task 17 (AGENTS.md) |
| 9. 理论来源 | Task 16 (README.md) |
| 10. 安全边界 | Task 17 (AGENTS.md) |
| Templates | Task 15 |

无遗漏。

**2. Placeholder 扫描：** 无 "TBD"、"TODO"、"implement later"。每个步骤都有具体操作和验证方式。

**3. 类型一致性：** 纯 Markdown 项目，无代码类型定义。所有文件路径在 Task 18 的验证清单中一一对应，保持唯一。
