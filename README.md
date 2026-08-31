# Postman

通用的群组/社区内容分发状态机，面向 ChatGPT Work、Codex 或其他具备浏览器操作能力的 Agent。

Postman 的核心是：**无人值守单轮批处理 + Excel 跨轮次持久化 + 入群与发帖双轨独立推进**。

```text
上传帖子 + Excel
    ↓
本轮无人值守自动处理
    ↓
每个群同时评估两条独立轨道
    ├─ Membership Lane：是否已加入 / 是否可以申请加入
    └─ Posting Lane：当前是否可以直接发帖
    ↓
哪个能推进就推进哪个
    ↓
每个动作后立即回写 Excel
    ↓
无法处理的状态记录后继续下一行
    ↓
返回更新后的 Excel
    ↓
下一轮重新上传，继续迭代
```

**人的介入点只在两轮之间，不在单次运行过程中。**

## 固定 Excel

固定七列，从左到右不得改名或换序：

| 列 | 字段 | 说明 |
|---|---|---|
| A | 序号 | 唯一行序号 |
| B | group名称 | 群组/社区名称 |
| C | group url | 目标群组 URL |
| D | 已加入 | `0/1`，只表示成员事实 |
| E | 入群状态 | 只描述成员/入群轨道 |
| F | 已发送 | `0/1`，只表示帖子是否已被平台接受 |
| G | 发送状态 | 只描述发帖轨道 |

模板：`postman_queue_template.xlsx`

## 最重要的设计：入群与发帖互相独立

禁止使用以下错误逻辑：

```text
已加入=0
  ↓
必须先加入
  ↓
没有加入就不检查能不能发
```

正确逻辑是：打开一个 group 后，同时判断两个问题：

1. **Membership Lane**：现在是不是成员？不是的话，能否正常申请加入？
2. **Posting Lane**：不管是不是成员，当前页面是否已经提供合法的发帖入口？

因此以下状态完全合法：

```text
已加入=0
入群状态=PENDING_APPROVAL
已发送=1
发送状态=SUCCESS_VISIBLE
```

含义：入群申请仍在审核，但该群允许非成员发帖，帖子已经成功发布。

另一种合法状态：

```text
已加入=0
入群状态=NEEDS_HUMAN:QUESTION_REQUIRES_USER_FACT
已发送=1
发送状态=SUCCESS_PENDING_REVIEW
```

含义：入群问题不能自动回答，但这不妨碍该群允许直接投稿。

**`已加入=0` 永远不能单独作为“不尝试发送”的理由。**

只有页面明确显示“必须成为成员后才能发帖”时，Posting Lane 才被 Membership Lane 阻塞。

## 无人值守模式

默认模式：

```text
UNATTENDED_BATCH
```

单次运行中 Agent 必须：

1. 不向用户提问；
2. 不逐群确认；
3. 不要求发帖前确认；
4. 不因为某一行失败而停止；
5. 不因为某一行需要人工信息而停止；
6. 不因为入群待审核而停止；
7. 入群失败后仍检查该群是否可直接发帖；
8. 发帖失败后仍允许 Membership Lane 继续完成；
9. 每个外部动作后立即回写 Excel；
10. 扫描全部本轮可自动处理的行；
11. 最终一次性返回更新后的 Excel。

## 两条独立状态轨道

### Membership Lane

`已加入`：

- `1`：确认已经是成员；
- `0`：当前没有确认成员身份。

常用 `入群状态`：

- `PENDING`
- `IN_PROGRESS`
- `SUCCESS`
- `SUCCESS:ALREADY_MEMBER`
- `PENDING_APPROVAL`
- `NOT_REQUIRED:POSTING_ALLOWED_WITHOUT_MEMBERSHIP`
- `FAILED:<reason>`
- `SKIPPED:<reason>`
- `NEEDS_HUMAN:<reason>`

`NOT_REQUIRED:*` 不等于已经加入，因此 `已加入` 仍为 `0`。

### Posting Lane

`已发送`：

- `1`：平台已经接受本次帖子提交；
- `0`：尚未确认提交成功。

常用 `发送状态`：

- `PENDING`
- `IN_PROGRESS`
- `SUCCESS_VISIBLE`
- `SUCCESS_PENDING_REVIEW`
- `BLOCKED:MEMBERSHIP_REQUIRED`
- `FAILED:<reason>`
- `SKIPPED:<reason>`
- `NEEDS_HUMAN:<reason>`

**等待管理员审核也必须记为 `已发送=1`**，防止下一轮重复提交。

## 单行推荐流程

```text
OPEN GROUP ONCE
  ↓
READ CURRENT PAGE STATE
  ↓
┌────────────────────────────┬────────────────────────────┐
│ Membership Lane            │ Posting Lane               │
│                            │                            │
│ D=1 → 不再申请加入          │ F=1 → 永不重复发           │
│ D=0 → 检查成员状态          │ F=0 → 检查当前发帖入口      │
│ 能申请 → 正常申请           │ 能直接发 → 直接发           │
│ 待审核 → PENDING_APPROVAL  │ 需成员 → BLOCKED           │
│ 问题无法答 → NEEDS_HUMAN   │ 可提交 → 提交并验证         │
└────────────────────────────┴────────────────────────────┘
  ↓
分别回写 D/E 与 F/G
  ↓
SAVE
  ↓
NEXT ROW
```

浏览器操作实际可以顺序点击，但**逻辑上两条 Lane 必须独立，不得人为串行依赖**。

## 跨轮次恢复

下一轮根据 Excel 继续：

- `已发送=1` → 永不再次发送；
- `PENDING_APPROVAL` → 自动复核成员审批；
- `BLOCKED:MEMBERSHIP_REQUIRED` → 先复核成员状态；如果已经加入，再重新尝试发送；
- `FAILED:*` → 临时失败可保守重试；
- `NEEDS_HUMAN:*` → 被动检查阻塞是否已在两轮之间解除；
- `IN_PROGRESS` → 先验证真实页面结果，再决定是否重试。

Excel 就是任务的持久化状态数据库。

## 核心不变量

```text
已加入=0  ≠  不能发帖
已加入=1  ≠  一定能发帖
已发送=1  ≠  已加入=1
入群失败   ≠  发帖失败
发帖失败   ≠  入群失败
```

只有平台明确要求 membership 时：

```text
Posting Lane --depends-on--> Membership Lane
```

否则两条轨道独立。

## 使用

### Codex / Skill

```text
.agents/skills/postman/SKILL.md
```

最短调用：

```text
Use $postman. Run unattended. Treat membership and posting as independent lanes. For every row, check both lanes and advance whichever is actionable. Do not require membership before posting unless the platform explicitly requires it. Checkpoint the Excel after every side effect and return the updated workbook.
```

### ChatGPT Work / 通用 Chat

使用 `PROMPT_CHATGPT.md`。

最短调用：

```text
按 Postman 无人值守双轨模式继续。入群和发帖分开判断、同步推进；即使已加入=0，也必须检查当前是否允许直接发帖。每个动作后回写 Excel；单行阻塞继续下一行；最终返回更新后的 Excel。
```

## 非目标

Postman 不负责绕过验证码、登录安全机制、群规、管理员审核、平台风控或账号限制，也不伪造入群问题答案。
