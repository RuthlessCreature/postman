# Postman

通用的群组/社区内容分发状态机，面向 ChatGPT Work、Codex 或其他具备浏览器操作能力的 Agent。

它解决的核心问题不是“写一段广告”，而是让 Agent 根据一个 Excel 队列持续迭代：**读取状态 → 处理下一行 → 执行入群/发帖 → 立即回写 Excel → 继续下一行**，从而避免重复入群、重复发帖、跑一两个群就丢状态。

## 核心输入

### 1. 内容

用户提供任意需要分发的内容/帖子，可直接粘贴给 Agent，也可以保存为 `content.md`。

### 2. Excel 队列

固定七列，从左到右不得改名或换序：

| 列 | 字段 | 说明 |
|---|---|---|
| A | 序号 | 唯一行序号 |
| B | group名称 | 群组/社区名称 |
| C | group url | 目标群组 URL |
| D | 已加入 | `0/1`；1=已确认具备发布所需成员资格 |
| E | 入群状态 | 详细状态与失败原因 |
| F | 已发送 | `0/1`；1=内容已被平台接受提交 |
| G | 发送状态 | 详细发布状态与失败原因 |

模板：`postman_queue_template.xlsx`

## 关键布尔语义

### 已加入

- `1`：确认已是成员，或本轮入群成功。
- `0`：尚未加入、申请待审核、失败、需要人工处理。

### 已发送

- `1`：平台已经接受本次内容提交，包括：
  - 已公开可见；
  - 已提交且等待管理员审核。
- `0`：尚未提交、提交失败、需要人工处理、被规则阻止。

这个定义很重要：**等待管理员审核也必须记为已发送=1**，否则下一轮会再次提交同一内容，造成重复发帖。

## 状态建议

### 入群状态

- `PENDING`
- `IN_PROGRESS`
- `SUCCESS`
- `SUCCESS:ALREADY_MEMBER`
- `PENDING_APPROVAL`
- `FAILED:<reason>`
- `SKIPPED:<reason>`
- `NEEDS_HUMAN:<reason>`

### 发送状态

- `PENDING`
- `IN_PROGRESS`
- `SUCCESS_VISIBLE`
- `SUCCESS_PENDING_REVIEW`
- `FAILED:<reason>`
- `SKIPPED:<reason>`
- `NEEDS_HUMAN:<reason>`

## 执行原则

1. Excel 是唯一事实源（Single Source of Truth）。
2. 每个外部动作完成后立即回写当前行，不等整批结束。
3. `已发送=1` 的行永远禁止再次发送，除非用户显式重置为 0。
4. `已加入=0` 时先处理成员资格；未取得成员资格时不得进入发送步骤。
5. 单行失败只记录并继续下一行，不得因为一个群组失败而终止整批。
6. CAPTCHA、登录验证、账号级限制等必须标记为 `NEEDS_HUMAN`/账号级阻塞，不得伪造成功。
7. 任何成功都必须基于页面可见证据，不得仅凭“点击过按钮”判定成功。
8. 不绕过平台规则、群规、管理员审核或账号限制。

## 推荐运行顺序

```text
LOAD_XLSX
  ↓
VALIDATE_SCHEMA
  ↓
FIND_NEXT_ROW
  ↓
CHECK_MEMBERSHIP
  ├─ 已加入 → UPDATE D/E → POST
  ├─ 入群成功 → UPDATE D/E → POST
  ├─ 待审批 → UPDATE D/E → NEXT_ROW
  └─ 失败/人工 → UPDATE D/E → NEXT_ROW
  ↓
POST_CONTENT
  ├─ 可见 → F=1, G=SUCCESS_VISIBLE
  ├─ 待审核 → F=1, G=SUCCESS_PENDING_REVIEW
  └─ 失败/人工 → F=0, G=FAILED/NEEDS_HUMAN
  ↓
SAVE_XLSX
  ↓
NEXT_ROW
```

## 使用方式

### Codex / Skill

项目级 Skill 位于：

```text
.agents/skills/postman/SKILL.md
```

### ChatGPT Work / 通用 Chat

直接使用：

```text
PROMPT_CHATGPT.md
```

上传/提供：

1. `postman_queue.xlsx`
2. 需要发布的内容
3. 如有必要，平台与浏览器使用说明

然后要求：

```text
按 Postman 状态机继续处理未完成行，处理过程中持续回写 Excel。
```

## 非目标

Postman 不负责：

- 绕过验证码或登录安全机制；
- 规避平台反垃圾或风控机制；
- 伪造入群问题答案；
- 在明确禁止发帖的社区中强行发布；
- 把失败操作伪装成成功。

## License

Internal / project use unless otherwise specified.
