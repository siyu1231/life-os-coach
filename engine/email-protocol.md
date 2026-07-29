# 企微邮件协议

本文件定义 Life Coach Agent 通过腾讯企业微信（WeCom）邮件 API 收发邮件的完整协议。涵盖：前置条件与鉴权、发送邮件、查询应用邮箱、轮询接收邮件、邮件正文模板规范、AI 工具实现指导。

## 前置条件

### 企微应用配置

使用本协议前，需在企微管理后台完成以下配置：

1. **创建自建应用**：在「应用管理」>「自建」中创建应用，获取 `AgentId` 和 `Secret`。
2. **开通邮箱权限**：在应用详情页的「企业邮箱」中，开通「发送邮件」和「读取邮件」权限。
3. **配置可信 IP**（可选但推荐）：在应用详情页配置服务器出口 IP 白名单，防止 `Secret` 泄露后被外部调用。
4. **记录 CorpId**：在「我的企业」>「企业信息」中获取 `CorpId`。

### access_token 获取

企微所有 API 调用均需在 URL 上携带 `access_token`。`access_token` 通过以下接口获取：

```text
GET https://qyapi.weixin.qq.com/cgi-bin/gettoken?corpid={CORP_ID}&corpsecret={SECRET}
```

**返回结构：**

```json
{
  "errcode": 0,
  "errmsg": "ok",
  "access_token": "accesstoken000001",
  "expires_in": 7200
}
```

**关键约束：**

| 属性 | 说明 |
|------|------|
| 有效期 | 7200 秒（2 小时） |
| 频率限制 | 2000 次/天，建议缓存复用，过期前 5 分钟主动刷新 |
| 并发安全 | 新 token 生成后旧 token 在 5 分钟内仍有效，不必担心正好过期导致短暂不可用 |

**缓存与刷新策略（伪代码）：**

```text
function get_access_token():
    if cache.access_token exists and cache.expires_at > now() + 300:
        return cache.access_token

    // 获取新 token（注意避免并发刷新）
    lock.acquire("token_refresh")
    try:
        // 双重检查：锁内再次确认缓存是否已被其他线程刷新
        if cache.access_token exists and cache.expires_at > now() + 300:
            return cache.access_token

        response = GET https://qyapi.weixin.qq.com/cgi-bin/gettoken
                   ?corpid=CORP_ID
                   &corpsecret=SECRET

        if response.errcode != 0:
            log_error("获取 access_token 失败", response)
            raise TokenError

        cache.access_token = response.access_token
        cache.expires_at = now() + response.expires_in  // 建议预留 300s 缓冲
        return cache.access_token
    finally:
        lock.release()
```

### 配置存储建议

相关凭证建议存储在 `profile/user.md` 的 `## 企微配置` 章节：

```markdown
## 企微配置

- `wecom.corp_id`: ww1234567890abcdef
- `wecom.agent_id`: 1000002
- `wecom.secret`: (通过环境变量 `WECOM_SECRET` 注入，不存明文)
```

## 发送邮件 API

### 接口地址

```text
POST https://qyapi.weixin.qq.com/cgi-bin/exmail/app/compose_send?access_token={ACCESS_TOKEN}
```

### 请求参数

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| `to.emails` | string[] | 否 | 收件人邮箱地址列表，与 `to.userids` 至少填一个 |
| `to.userids` | string[] | 否 | 收件人企微 UserId 列表，与 `to.emails` 至少填一个 |
| `cc` | string[] | 否 | 抄送人邮箱地址列表 |
| `bcc` | string[] | 否 | 密送人邮箱地址列表 |
| `subject` | string | **是** | 邮件主题（标题） |
| `content` | string | **是** | 邮件正文，HTML 或纯文本 |
| `content_type` | string | 否 | 正文类型：`"html"`（默认）或 `"text"` |
| `attachment_list` | object[] | 否 | 附件列表，每个附件含 `file_name`（文件名）、`file_content`（base64 编码内容） |

**请求体示例（JSON）：**

```json
{
  "to": {
    "emails": ["user@example.wecom.work"],
    "userids": ["zhangsan"]
  },
  "subject": "下午好——今天进展怎么样？",
  "content": "<p>下午好！</p><p>今天下午最重要的 1-2 件事有推进吗？</p>",
  "content_type": "html"
}
```

### 响应结构

**成功响应：**

```json
{
  "errcode": 0,
  "errmsg": "ok"
}
```

**常见错误码：**

| errcode | 含义 | 处理建议 |
|---------|------|---------|
| 0 | 成功 | — |
| 40001 | access_token 过期或无效 | 重新获取 access_token 后重试 |
| 41001 | 缺少 access_token 参数 | 检查请求 URL |
| 42001 | access_token 超时 | 重新获取后重试 |
| 43002 | 需要 POST 方法 | 检查 HTTP 方法 |
| 60001 | 部门不存在 | 检查请求参数 |
| 84001 | 邮件功能未开通 | 在企微管理后台开通企业邮箱权限 |
| 84002 | 发件人邮箱未配置 | 在应用详情页配置发件邮箱 |
| 84003 | 收件人邮箱地址无效 | 检查邮箱地址格式 |
| 84004 | 邮件正文为空 | 检查 `content` 字段 |
| 84005 | 邮件主题为空 | 检查 `subject` 字段 |

### 发送后的异步跟进

发送邮件后，Agent 应执行与 IM 消息相同的动态跟进逻辑（详见 `engine/cron-system.md` 的「发送后动态 Timer 实现说明」章节）：

1. 记录本次发送的 `last_sent_message_id`（可使用邮件 Message-ID 或企微返回的内部 ID）和 `last_sent_timestamp`。
2. 如果 `cron.followup_5min_enabled` 为 `true`，在发送后 5 分钟检查用户是否回复。
3. 如果 `cron.followup_10min_enabled` 为 `true`，在发送后 10 分钟检查用户是否回复，未回复则归档为「等待用户回复」。

邮件渠道的「是否回复」判断：在发送后 5 分钟（或 10 分钟）内，轮询收件箱时如果发现来自收件人的新邮件（发件人地址匹配、时间在发送后的窗口内），视为用户已回复。

## 查询应用邮箱 API

### 接口地址

```text
POST https://qyapi.weixin.qq.com/cgi-bin/exmail/app/get_email_alias?access_token={ACCESS_TOKEN}
```

### 请求体

```json
{}
```

（此接口不需要请求参数，传空 JSON 对象即可。）

### 响应结构

```json
{
  "errcode": 0,
  "errmsg": "ok",
  "email": "life-coach@yourcompany.wecom.work",
  "alias_list": [
    "coach@yourcompany.wecom.work",
    "lifecoach@yourcompany.wecom.work"
  ]
}
```

**返回字段说明：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `email` | string | 应用的主邮箱地址，发件人地址即为此值 |
| `alias_list` | string[] | 该邮箱的别名列表，所有别名均可用于接收邮件（发往别名 = 发往主邮箱） |

### 使用时机

在 Agent 初始化阶段调用一次此接口，将返回值缓存。用途：

- 发件时确认发件人地址。
- 收件时用于判断用户收件人字段（用户回复邮件时，To 字段包含此地址或其别名之一）。

## 接收邮件策略

### 为什么用轮询而非回调

企微企业邮箱的开放平台支持邮件回调通知，但**回调仅推送邮件数量（新邮件到达事件），不推送邮件正文**。因此完整的「接收阅读」链路必须通过 API 拉取，分为两步：

1. **获取收件箱列表**：获取邮件摘要列表（发件人、主题、时间、邮件 ID）。
2. **获取邮件内容**：根据邮件 ID 拉取完整正文。

### 整体轮询流程

```text
function poll_for_new_mails():
    access_token = get_access_token()

    // 第 1 步：获取收件箱列表
    mail_list = fetch_inbox_list(access_token, since=last_poll_time)

    // 第 2 步：对新邮件逐封获取正文
    for each mail in mail_list:
        if mail.id in already_processed_ids:
            continue
        full_mail = fetch_mail_content(access_token, mail.id)
        process_mail(full_mail)          // 解析意图 → 交给 session-flow
        already_processed_ids.add(mail.id)

    // 更新轮询游标
    last_poll_time = now()
```

### 第 1 步：获取收件箱列表 API

**接口地址（标准企微邮箱 API）：**

```text
POST https://qyapi.weixin.qq.com/cgi-bin/exmail/app/list_mail?access_token={ACCESS_TOKEN}
```

**请求参数：**

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| `mail_type` | int | 是 | 邮箱类型：`1` = 收件箱 |
| `limit` | int | 否 | 每页条数，默认 20，最大 50 |
| `offset` | int | 否 | 偏移量，从 0 开始 |
| `begin_time` | int | 否 | 起始时间，Unix 时间戳（秒）。用于按时间段筛选，避免全量拉取 |
| `end_time` | int | 否 | 结束时间，Unix 时间戳（秒） |

**请求体示例：**

```json
{
  "mail_type": 1,
  "limit": 20,
  "begin_time": 1753613400,
  "end_time": 1753699800
}
```

**响应结构：**

```json
{
  "errcode": 0,
  "errmsg": "ok",
  "total": 3,
  "mail_list": [
    {
      "mailid": "abc123def456",
      "subject": "Re: 下午好——今天进展怎么样？",
      "from": "zhangsan@yourcompany.wecom.work",
      "send_time": 1753699000,
      "has_attachment": false,
      "is_read": false
    }
  ]
}
```

### 第 2 步：获取邮件内容 API

**接口地址（标准企微邮箱 API）：**

```text
POST https://qyapi.weixin.qq.com/cgi-bin/exmail/app/get_mail_content?access_token={ACCESS_TOKEN}
```

**请求参数：**

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| `mailid` | string | **是** | 邮件 ID，从列表接口返回 |
| `type` | string | 否 | 返回内容格式：`"html"`（默认）或 `"text"` |

**请求体示例：**

```json
{
  "mailid": "abc123def456",
  "type": "text"
}
```

**响应结构：**

```json
{
  "errcode": 0,
  "errmsg": "ok",
  "mailid": "abc123def456",
  "subject": "Re: 下午好——今天进展怎么样？",
  "from": "zhangsan@yourcompany.wecom.work",
  "to": ["life-coach@yourcompany.wecom.work"],
  "cc": [],
  "send_time": 1753699000,
  "content": "下午好！下午推进了设计稿，基本完成。不过下午 3 点有点犯困，效率不高。",
  "attachment_list": [],
  "reply_to_mailid": "xyz789ghi012"
}
```

**返回字段说明：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `content` | string | 邮件正文（已按 `type` 参数格式化，纯文本或 HTML） |
| `from` | string | 发件人邮箱 |
| `to` | string[] | 收件人列表 |
| `reply_to_mailid` | string | 如果是回复邮件，此字段指向被回复的原邮件 ID，可用于构建对话线索 |

### 轮询策略与频率

| 场景 | 轮询间隔 | 说明 |
|------|---------|------|
| Agent 刚发送邮件后 | 30 秒/次，持续 15 分钟 | 密集轮询用于快速感知用户回复，支持 +5min / +10min 跟进逻辑 |
| 常规待机状态 | 5 分钟/次 | 不需要即时的场景，降低 API 调用量 |
| 非工作时间（休息日/夜间） | 10 分钟/次 | 进一步降低调用频率 |

**伪代码：**

```text
function get_poll_interval():
    if within_15min_after_sending():
        return 30_seconds
    if is_night_time() or not is_workday():
        return 10_minutes
    return 5_minutes
```

### 去重与幂等

使用本地已处理邮件 ID 集合（`already_processed_ids`）确保同一封邮件不会被重复处理。该集合可持久化到 `system/state/mail-cursor.json`：

```json
{
  "last_poll_time": 1753699800,
  "last_processed_mailid": "abc123def456",
  "processed_mailids": ["abc123def456", "ghi789jkl012"]
}
```

文件大小控制：`processed_mailids` 保留最近 500 条，超出时按时间戳裁剪最早的一半。

## 邮件正文模板

### 模板结构设计原则

Agent 发送的每封邮件遵循统一的模板结构，确保用户一眼就能理解上下文和期望的回复方式：

1. **时间点卡片**：一个清晰的标记告知用户这是哪个时间点的触达（如「09:30 早安问候」「14:30 午后启动」），让用户建立节奏预期。
2. **上下文简述**：1-2 句话说明当前时间背景和发送缘由，降低认知负担（不做无上下文的信息轰炸）。
3. **具体问题**：1 个核心问题，聚焦于当前时间点的目标（规划、检查、复盘等）。
4. **行动提示**：给出 2-3 个可选的简短回复方式（选项而非开放式提问），或一个明确的下一步建议。

### 结构化模板代码（HTML）

以下为通用 HTML 邮件模板，各时间点通过替换变量内容生成具体邮件：

```html
<div style="max-width:600px;margin:0 auto;font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;color:#333;">
  <!-- 时间点卡片 -->
  <div style="background:#f0f7ff;border-left:4px solid #3b82f6;padding:12px 16px;margin-bottom:16px;border-radius:0 4px 4px 0;">
    <span style="font-size:14px;color:#3b82f6;font-weight:600;">{{TIME_LABEL}}</span>
  </div>

  <!-- 上下文简述 -->
  <p style="font-size:15px;line-height:1.6;margin:0 0 12px 0;">{{CONTEXT_INTRO}}</p>

  <!-- 核心问题 -->
  <p style="font-size:15px;line-height:1.6;margin:0 0 16px 0;font-weight:600;">{{CORE_QUESTION}}</p>

  <!-- 行动提示 -->
  <div style="background:#f9fafb;padding:12px 16px;border-radius:4px;">
    <p style="font-size:13px;color:#6b7280;margin:0 0 8px 0;">直接回复此邮件即可：</p>
    <ul style="margin:0;padding-left:20px;font-size:14px;color:#374151;">
      {{OPTIONS_LIST}}
    </ul>
  </div>

  <!-- 页脚签名 -->
  <div style="margin-top:24px;padding-top:16px;border-top:1px solid #e5e7eb;font-size:12px;color:#9ca3af;">
    <p style="margin:0;">Life Coach Agent</p>
    <p style="margin:4px 0 0 0;">每天几个时间点，陪你关注最重要的事。</p>
    <p style="margin:4px 0 0 0;">如不想收到此类邮件，回复「暂停邮件」。</p>
  </div>
</div>
```

### 各时间点变量填充表

以下为各时间点的模板变量内容。每个时间点的 `CORE_QUESTION` 和 `OPTIONS_LIST` 经过设计，满足「一个核心问题 + 提供选项而非开放提问」的约束。

**早安问候（09:30 工作日）：**

| 变量 | 值 |
|------|---|
| `TIME_LABEL` | 09:30 早安问候 |
| `CONTEXT_INTRO` | 早上好！新的一天开始了，先花1分钟看清今天的局面。 |
| `CORE_QUESTION` | 今天最重要的 1 件事是什么？ |
| `OPTIONS_LIST` | `<li>今天主要推进 [某项目]，目标是在 [某时间] 完成 [具体产出]</li><li>今天有 [N] 件事要处理，优先级最高的是 [这件事]</li><li>今天精力一般，先稳住节奏，做 [这件可控的事]</li>` |

**上午状态检查（10:30 工作日）：**

| 变量 | 值 |
|------|---|
| `TIME_LABEL` | 10:30 上午状态检查 |
| `CONTEXT_INTRO` | 上午已进行 1 小时了，来一个快速 pulse check。 |
| `CORE_QUESTION` | 进度和精力怎么样？ |
| `OPTIONS_LIST` | `<li>进展顺利，按计划推进中</li><li>有点卡住了，卡在 [这里]</li><li>精力有点低，可能需要调整一下节奏</li>` |

**午后启动引导（14:30 工作日）：**

| 变量 | 值 |
|------|---|
| `TIME_LABEL` | 14:30 午后启动 |
| `CONTEXT_INTRO` | 午休结束，下午的精力高峰即将到来。 |
| `CORE_QUESTION` | 下午最重要的 1-2 件事是什么？ |
| `OPTIONS_LIST` | `<li>下午要完成 [这件事]，预计 [时间] 搞定</li><li>下午有个 [会议/讨论]，需要提前准备 [这些]</li><li>下午精力预计不高，先做 [这件轻松但重要的事]</li>` |

**下午中段检查（15:30 工作日）：**

| 变量 | 值 |
|------|---|
| `TIME_LABEL` | 15:30 下午中段状态检查 |
| `CONTEXT_INTRO` | 下午过半，快速检查一下进度和精力状态。 |
| `CORE_QUESTION` | 需要调整节奏或切换任务类型吗？ |
| `OPTIONS_LIST` | `<li>节奏刚好，继续推进中</li><li>有点累了，换个轻松的任务做做</li><li>效率不太高，可能需要起来走动一下</li>` |

**收尾预备（16:30 工作日）：**

| 变量 | 值 |
|------|---|
| `TIME_LABEL` | 16:30 收尾预备 |
| `CONTEXT_INTRO` | 距离日终还有 1 小时，看看还有什么需要推进的。 |
| `CORE_QUESTION` | 今天还有什么必须在结束前推进的？ |
| `OPTIONS_LIST` | `<li>还有 [这件事] 想推进一点，目标是在 [时间] 前完成 [这部分]</li><li>重要的事都推进了，剩下的明天再说</li><li>今天差不多了，开始收尾整理</li>` |

**日终收尾（17:30 工作日）：**

| 变量 | 值 |
|------|---|
| `TIME_LABEL` | 17:30 日终收尾 |
| `CONTEXT_INTRO` | 一天的节奏接近尾声，花 2 分钟做个轻量回顾。 |
| `CORE_QUESTION` | 今天完成了什么？有什么偏离计划的地方？ |
| `OPTIONS_LIST` | `<li>今天完成了 [这些]，整体在计划内</li><li>今天 [这件事] 没按计划推进，原因是 [这个]</li><li>今天感觉 [不错/一般/有点累]，主要收获是 [这个]</li>` |

**晚间复盘（22:30 每日）：**

| 变量 | 值 |
|------|---|
| `TIME_LABEL` | 22:30 晚间复盘 |
| `CONTEXT_INTRO` | 夜深了，花 1 分钟回顾今天。 |
| `CORE_QUESTION` | 今天有没有一个让你觉得「今天没白过」的瞬间？ |
| `OPTIONS_LIST` | `<li>有的，今天 [这件事] 让我觉得充实</li><li>说不上有高光时刻，但整体平稳</li><li>今天有点难，但过去了</li>` |

**晚间复盘——休息日版（22:30 周末/节假日）：**

| 变量 | 值 |
|------|---|
| `TIME_LABEL` | 22:30 晚间回顾 |
| `CONTEXT_INTRO` | 不打扰白天，只在睡前留一个轻量回声。 |
| `CORE_QUESTION` | 今天休息得怎么样？ |
| `OPTIONS_LIST` | `<li>休息得不错，[这件事] 让我放松了一下</li><li>还行，明天想试试 [这件小事]</li><li>晚安就好</li>` |

### 模板变量实现接口

以上模板变量供 `engine/session-flow.md` 中的会话流逻辑调用。具体实现时，每个时间点对应一个变量字典，由 session-flow 组装后传入邮件发送函数。

```text
// 伪代码：组装并发送邮件
function send_cron_email(time_slot: str, user_profile: dict):
    vars = get_email_vars(time_slot, user_profile)     // 从模板变量表中读取
    subject = build_subject(vars.TIME_LABEL)            // 主题 = 时间点标签
    html_body = render_template("email_template.html", vars)
    content = truncate_to_limit(html_body, 200)         // 正文约束检查
    send_mail(to=user_profile.email, subject=subject, content=content, content_type="html")
```

## 正文约束

### 字数限制

邮件正文（不含 HTML 标签的纯文本内容）**不超过 200 字**。此限制基于邮件场景的阅读体验：

- 用户可能用手机邮件客户端查看，长文本阅读体验差。
- 邮件不是即时通讯，过长的邮件容易被标记为「稍后看」然后遗忘。
- 200 字足够表达一个核心问题和选项，同时保持简洁。

**实现约束检查：**

```text
function truncate_to_limit(html_content, max_chars=200):
    plain_text = strip_html_tags(html_content)
    if char_count(plain_text) <= max_chars:
        return html_content
    // 超出时截断最后一个完整句子，加省略号
    truncated = plain_text.truncate_to(max_chars - 1)
    return "<p>" + truncated + "…</p>"
```

### 每封邮件的核心问题原则

1. **一封邮件一个问题**：不同时间触及不同核心问题（规划、检查、复盘等），每封邮件只聚焦一个。
2. **给选项优于开放式提问**：每个问题的行动提示部分提供 2-3 个具体可选的回复方向，降低用户的回复心理门槛。与其问「今天怎么样？」，不如问「今天完成了 [这件事] 还是还在推进中？」。
3. **不追问**：如果用户未回复上一封邮件，下一封邮件不追问「上次的问题你还没回答」。每封邮件独立成文，不依赖历史上下文（但 Agent 内部维护上下文）。
4. **不评价**：不对用户的行为或回复内容做价值判断（如「你今天只完成了这个？」）。保持好奇、中立的语调。
5. **不连续发送**：如果上一封邮件发送后 10 分钟内用户未回复且已被归档为「等待用户回复」，同一天内不再通过邮件渠道发送新的主动触达（除非用户主动回复「继续」或手动触发）。

### 邮件主题规范

| 时间点 | 主题格式 | 示例 |
|--------|---------|------|
| 早安问候 | `{TIME_LABEL}` | `09:30 早安问候` |
| 状态检查 | `{TIME_LABEL}` | `10:30 上午状态检查` |
| 复盘类 | `{TIME_LABEL}` | `22:30 晚间复盘` |

主题直接使用时间点标签，保持简洁一致。不使用自定义标题，以确保用户收件箱中来自 Agent 的邮件主题可预测、可筛选。

## 接收邮件后的处理流程

当通过轮询获取到用户回复邮件后，处理流程如下：

```text
function process_mail(mail_content):
    // 1. 关联上下文
    parent_mail = null
    if mail_content.reply_to_mailid exists:
        parent_mail = get_sent_mail_by_id(mail_content.reply_to_mailid)
        // parent_mail 包含发送时的时间点标签（TIME_LABEL），
        // 帮助 Agent 理解用户当前是在回复哪封触达邮件

    // 2. 提取纯文本内容
    plain_text = strip_html_tags(mail_content.content).trim()

    // 3. 判断特殊指令
    if plain_text contains "暂停邮件" or plain_text contains "停止邮件":
        set_cron_enabled(false)
        send_confirmation("已暂停邮件触达。随时回复「恢复邮件」重新开启。")
        return

    if plain_text contains "恢复邮件" or plain_text contains "开启邮件":
        set_cron_enabled(true)
        send_confirmation("已恢复邮件触达。接下来的时间点我会继续发邮件。")
        return

    // 4. 交给 session-flow 处理
    session_flow.handle_user_message(
        channel="email",
        content=plain_text,
        context={
            "time_label": parent_mail.time_label if parent_mail else null,
            "mailid": mail_content.mailid,
            "from": mail_content.from
        }
    )
```

### 特殊指令集

用户可通过邮件回复中的关键词控制 Agent 行为。以下指令不区分大小写，支持中文指令：

| 指令关键词 | 作用 | Agent 回复 |
|-----------|------|-----------|
| `暂停邮件` / `停止邮件` | 暂停所有邮件触达，等同于 `cron.enabled = false` | 确认暂停，提示恢复方式 |
| `恢复邮件` / `开启邮件` | 恢复邮件触达，等同于 `cron.enabled = true` | 确认恢复 |
| `暂停一天` / `今天暂停` | 仅暂停今日剩余时间点的邮件触达，明天自动恢复 | 确认暂停，提示明天恢复 |
| `总结` / `本周总结` | 触发生成本周交互摘要（非实时回复，在下一个时间点或 5 分钟内发送） | 提示正在生成 |

## AI 工具实现指导

### 通用架构说明

邮件收发组件是 Life Coach Agent 的一个独立 I/O 通道模块，与 IM 通道（企微消息、飞书消息等）并列。核心职责：

1. **邮件发送**：将 session-flow 生成的消息通过企微邮件 API 发出。
2. **邮件接收**：通过轮询获取用户回复，提取文本内容后交给 session-flow 统一处理。

### 通道抽象

推荐将邮件通道与 IM 通道抽象为统一的 `MessageChannel` 接口：

```text
interface MessageChannel:
    send(to, content, context) -> message_id
    poll(since_timestamp) -> list[message]
    get_channel_type() -> "email" | "im" | "other"
```

邮件通道的实现：

```text
class EmailChannel implements MessageChannel:
    def send(to, content, context):
        // 1. 加载用户邮箱配置
        // 2. 从模板变量表获取对应 TIME_LABEL 的 subject 和 content
        // 3. 调用 compose_send API
        // 4. 返回企微返回的 mailid 作为 message_id
        // 5. 记录到已发送邮件日志

    def poll(since_timestamp):
        // 1. 调用 list_mail API 获取 since 之后的新邮件
        // 2. 对每封新邮件调用 get_mail_content API 获取正文
        // 3. 返回统一的消息结构列表
        // 4. 更新轮询游标

    def get_channel_type():
        return "email"
```

### Claude Code 中的集成

如果 Life Coach Agent 运行在 Claude Code 环境中：

**邮件发送工具定义（MCP 工具或 Code 函数）：**

```text
// MCP tool: send_wecom_email
{
  "name": "send_wecom_email",
  "description": "通过企微邮箱 API 发送邮件",
  "inputSchema": {
    "type": "object",
    "properties": {
      "to": {"type": "array", "items": {"type": "string"}, "description": "收件人邮箱列表"},
      "subject": {"type": "string", "description": "邮件主题"},
      "content": {"type": "string", "description": "HTML 邮件正文"},
      "cc": {"type": "array", "items": {"type": "string"}, "description": "抄送列表"},
      "bcc": {"type": "array", "items": {"type": "string"}, "description": "密送列表"}
    },
    "required": ["to", "subject", "content"]
  }
}

// MCP tool: poll_wecom_emails
{
  "name": "poll_wecom_emails",
  "description": "轮询企微邮箱获取新邮件",
  "inputSchema": {
    "type": "object",
    "properties": {
      "since_minutes": {"type": "integer", "description": "获取最近 N 分钟内的新邮件"}
    }
  }
}
```

**Claude Code 定时轮询注册：**

```text
// 初始化时注册邮件轮询定时任务
/loop 0/5 * * * * 轮询新邮件——调用 poll_wecom_emails 获取最近 5 分钟的邮件并处理回复
```

**settings.json 中的 MCP 配置示例：**

```json
{
  "mcpServers": {
    "wecom-mail": {
      "type": "stdio",
      "command": "node",
      "args": ["./integrations/wecom-mail-server.js"],
      "env": {
        "WECOM_CORP_ID": "ww1234567890abcdef",
        "WECOM_SECRET": "${env:WECOM_SECRET}",
        "WECOM_POLL_INTERVAL_MS": "300000"
      }
    }
  }
}
```

### 自建邮件服务（Node.js 示例骨架）

如果自建邮件收发服务，以下为 Node.js 实现骨架：

```javascript
// integrations/wecom-mail-server.js —— 骨架代码，仅示意关键流程
const axios = require('axios');

const BASE_URL = 'https://qyapi.weixin.qq.com/cgi-bin/exmail/app';
const TOKEN_URL = 'https://qyapi.weixin.qq.com/cgi-bin/gettoken';

let tokenCache = { token: null, expiresAt: 0 };

async function getAccessToken() {
  if (tokenCache.token && Date.now() < tokenCache.expiresAt - 300_000) {
    return tokenCache.token;
  }
  const resp = await axios.get(TOKEN_URL, {
    params: { corpid: process.env.WECOM_CORP_ID, corpsecret: process.env.WECOM_SECRET }
  });
  tokenCache.token = resp.data.access_token;
  tokenCache.expiresAt = Date.now() + resp.data.expires_in * 1000;
  return tokenCache.token;
}

async function sendEmail({ to, cc, bcc, subject, content, content_type = 'html' }) {
  const token = await getAccessToken();
  const resp = await axios.post(`${BASE_URL}/compose_send?access_token=${token}`, {
    to: { emails: to },
    cc: cc || [],
    bcc: bcc || [],
    subject,
    content,
    content_type
  });
  return resp.data;
}

async function getEmailAlias() {
  const token = await getAccessToken();
  const resp = await axios.post(`${BASE_URL}/get_email_alias?access_token=${token}`, {});
  return resp.data;
}

async function pollNewMails(sinceTimestamp) {
  const token = await getAccessToken();
  const resp = await axios.post(`${BASE_URL}/list_mail?access_token=${token}`, {
    mail_type: 1,
    limit: 50,
    begin_time: sinceTimestamp
  });
  return resp.data.mail_list || [];
}

async function getMailContent(mailid, type = 'text') {
  const token = await getAccessToken();
  const resp = await axios.post(`${BASE_URL}/get_mail_content?access_token=${token}`, {
    mailid,
    type
  });
  return resp.data;
}

module.exports = { sendEmail, getEmailAlias, pollNewMails, getMailContent };
```

### Python 实现示例骨架

```python
# integrations/wecom_mail.py —— 骨架代码，仅示意关键流程
import time, requests, os

BASE_URL = "https://qyapi.weixin.qq.com/cgi-bin/exmail/app"
TOKEN_URL = "https://qyapi.weixin.qq.com/cgi-bin/gettoken"

_token_cache = {"token": None, "expires_at": 0}

def get_access_token():
    if _token_cache["token"] and time.time() < _token_cache["expires_at"] - 300:
        return _token_cache["token"]
    resp = requests.get(TOKEN_URL, params={
        "corpid": os.environ["WECOM_CORP_ID"],
        "corpsecret": os.environ["WECOM_SECRET"],
    })
    data = resp.json()
    _token_cache["token"] = data["access_token"]
    _token_cache["expires_at"] = time.time() + data["expires_in"]
    return _token_cache["token"]

def send_email(to, subject, content, cc=None, bcc=None, content_type="html"):
    token = get_access_token()
    body = {
        "to": {"emails": to},
        "cc": cc or [],
        "bcc": bcc or [],
        "subject": subject,
        "content": content,
        "content_type": content_type,
    }
    resp = requests.post(f"{BASE_URL}/compose_send?access_token={token}", json=body)
    return resp.json()

def get_email_alias():
    token = get_access_token()
    resp = requests.post(f"{BASE_URL}/get_email_alias?access_token={token}", json={})
    return resp.json()

def poll_new_mails(since_timestamp):
    token = get_access_token()
    resp = requests.post(f"{BASE_URL}/list_mail?access_token={token}", json={
        "mail_type": 1,
        "limit": 50,
        "begin_time": since_timestamp,
    })
    return resp.json().get("mail_list", [])

def get_mail_content(mailid, type="text"):
    token = get_access_token()
    resp = requests.post(f"{BASE_URL}/get_mail_content?access_token={token}", json={
        "mailid": mailid,
        "type": type,
    })
    return resp.json()
```

## 与其他 engine 模块的关系

- **session-flow.md**：读取本文件的邮件模板变量表，在每个 Cron 触发时将生成的消息通过邮件通道发送；邮件通道收到用户回复后，提取文本交给 session-flow 统一处理。
- **cron-system.md**：邮件通道是 Cron 触发的消息分发渠道之一。发送后 +5min/+10min 的跟进逻辑同样适用于邮件渠道（通过轮询判断用户是否回复）。
- **integrations/tools.md**：企微应用凭证（CorpId、AgentId、Secret）的配置位置；自建邮件服务的部署说明也在此维护。
- **profile/user.md**：用户邮箱地址、邮件触达偏好（暂停/恢复、频率调整）的存放位置。

## 文件完整性验证

### API 引用检查

运行以下命令确认本文件覆盖了所有核心 API：

```bash
grep -c "compose_send\|get_email_alias" engine/email-protocol.md
```

预期：至少匹配 2。

### 鉴权引用检查

```bash
grep -c "access_token\|ACCESS_TOKEN" engine/email-protocol.md
```

预期：至少匹配 1。

### 接收策略覆盖检查

```bash
grep -c "轮询\|回调\|接收" engine/email-protocol.md
```

预期：至少匹配 2。

## References

- `engine/cron-system.md`：Cron 调度规则，邮件通道的上游触发源。
- `engine/session-flow.md`：会话流控制，邮件通道的消息生产者和消费者。
- `integrations/tools.md`：企微凭证和邮件服务部署说明。
- `profile/user.md`：用户邮箱地址和邮件偏好配置。
