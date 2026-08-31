# Postman

通用的群组/社区内容分发状态机，面向 ChatGPT Work、Codex 或其他具备浏览器操作能力的 Agent。

它解决的核心问题不是“帮我发几个帖子”，而是让 Agent 根据一个 Excel 队列进行**无人值守单轮批处理 + 跨轮次持续迭代**：

```text
上传帖子 + Excel
    ↓
本轮无人值守自动处理
    ↓
每行完成后回写 Excel
    ↓
无法处理的行记录状态后跳过
    ↓
继续处理剩余行
    ↓
本轮结束返回更新后的 Excel
    ↓
下次重新上传该 Excel
    ↓
从现有状态继续迭代
```

**人的介入点只在两轮之间，不在单次运行过程中。**

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

## 无人值守模式

默认运行模式为：

```text
UNATTENDED_BATCH
```

单次运行中 Agent 必须：

1. 不向用户提问；
2. 不要求逐群确认；
3. 不要求发帖前确认；
4. 不因为某一行失败而停止；
5. 不因为某一行需要人工信息而停止；
6. 不因为某一群入群待审核而停止；
7. 每个外部动作完成后立即回写 Excel；
8. 扫描并处理所有本轮可自动处理的行；
9. 最终一次性返回更新后的 Excel 和结果摘要。

如果某行需要人工处理，写：

```text
NEEDS_HUMAN:<原因>
```

然后**继续下一行**，而不是停下来叫用户处理。

只有登录/设备验证、账号级限制、平台级 CAPTCHA、浏览器整体失效等导致整个会话无法继续时，才提前结束本轮；结束前仍必须保存 Excel。

## 关键布尔语义

### 已加入

- `1`：确认已是成员，或本轮入群成功。
- `0`：尚未加入、申请待审核、失败或暂时不能自动完成。

### 已发送

- `1`：平台已经接受本次内容提交，包括：
  - 已公开可见；
  - 已提交且等待管理员审核。
- `0`：尚未确认提交成功。

**等待管理员审核也必须记为 `已发送=1`**，否则下一轮可能重复提交。

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

详细定义见 `SCHEMA.md`。

## 跨轮次恢复

下一轮拿到上一轮更新后的 Excel 后，Agent 不得从头盲跑，而应：

- `已发送=1` → 永久跳过发送；
- `PENDING_APPROVAL` → 自动检查是否已入群；
- `FAILED:*` → 临时失败可在后续轮次保守重试；
- `NEEDS_HUMAN:*` → 只被动检查阻塞是否已经在两轮之间解除；
- `IN_PROGRESS` → 先验证真实页面状态，再决定是否重试，防止重复发帖。

因此 Excel 本身就是整个任务的持久化状态数据库。

## 核心执行原则

1. Excel 是唯一事实源（Single Source of Truth）。
2. 每个外部动作完成后立即回写当前行，不等整批结束。
3. `已发送=1` 的行永远禁止再次发送，除非用户在两轮之间显式重置为0。
4. `已加入=0` 时先处理成员资格；未取得成员资格且非成员不能发帖时，记录状态后继续下一行。
5. 单行失败只记录并继续下一行，不得终止整批。
6. 无法回答的入群问题只标记 `NEEDS_HUMAN`，不得中途询问用户。
7. 任何成功都必须基于页面可见证据，不得仅凭“点击过按钮”判定成功。
8. 不绕过平台规则、群规、管理员审核、验证码或账号限制。

## 推荐运行顺序

```text
LOAD_XLSX
  ↓
VALIDATE_SCHEMA
  ↓
RECOVERY_SWEEP
  ↓
FOR EACH ROW
  ├─ 已发送=1 → NEXT
  ├─ 检查/恢复成员状态
  ├─ 能入群 → 入群 → 回写
  ├─ 不能自动入群 → 记状态 → NEXT
  ├─ 能发帖 → 发帖 → 验证 → 回写
  └─ 不能自动发帖 → 记状态 → NEXT
  ↓
SAVE_XLSX
  ↓
RETURN UPDATED XLSX
```

## 使用方式

### Codex / Skill

项目级 Skill：

```text
.agents/skills/postman/SKILL.md
```

最短调用：

```text
Use $postman. Run in unattended batch mode. Process every currently actionable row, never ask me questions during the run, checkpoint the Excel after each action, and return the updated workbook for the next iteration.
```

### ChatGPT Work / 通用 Chat

使用：

```text
PROMPT_CHATGPT.md
```

提供：

1. `postman_queue.xlsx`
2. 需要发布的内容
3. 必要的平台/浏览器说明

最短调用：

```text
按 Postman 无人值守批处理模式继续迭代。不要中途问我问题；处理所有本轮可处理行，每个动作后立即回写 Excel；单行阻塞记录后继续；只有整个会话无法继续时才提前结束，并返回更新后的 Excel。
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
