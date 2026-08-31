# Postman Excel Schema

## Sheet

默认工作表名建议为 `Groups`。用户现有文件使用其他工作表名也可以，只要完整包含固定七列。

## Fixed columns

| Col | Header | Type | Allowed values / format |
|---|---|---|---|
| A | 序号 | Integer | `>=1`，建议唯一 |
| B | group名称 | Text | 非空 |
| C | group url | Text/URL | 目标群组 URL |
| D | 已加入 | Integer Boolean | `0` or `1` |
| E | 入群状态 | Text | 状态码，可追加失败原因 |
| F | 已发送 | Integer Boolean | `0` or `1` |
| G | 发送状态 | Text | 状态码，可追加失败原因 |

这七列既是输入，也是跨轮次持久化状态。**不得依赖聊天记忆代替 Excel。**

## Membership state mapping

| 入群状态 | 已加入 | Meaning | Next-run behavior |
|---|---:|---|---|
| `PENDING` | 0 | 尚未处理 | 自动尝试 |
| `IN_PROGRESS` | 0 | 上轮可能中断 | 先验证真实状态再重试 |
| `SUCCESS` | 1 | 加入成功 | 不再入群 |
| `SUCCESS:ALREADY_MEMBER` | 1 | 页面确认原本已加入 | 不再入群 |
| `PENDING_APPROVAL` | 0 | 入群申请已提交 | 下轮自动复核，不盲目重复申请 |
| `FAILED:<reason>` | 0 | 入群失败 | 临时失败可在后续轮次再试一次 |
| `SKIPPED:<reason>` | 0 | 规则/目标原因主动跳过 | 默认不重试 |
| `NEEDS_HUMAN:<reason>` | 0 | 本轮不能自动解决 | 本轮跳过；下轮可被动复核是否已变化 |

## Sending state mapping

| 发送状态 | 已发送 | Meaning | Next-run behavior |
|---|---:|---|---|
| `PENDING` | 0 | 尚未处理 | 自动尝试 |
| `IN_PROGRESS` | 0 | 上轮可能在提交附近中断 | 必须验证是否已经提交，禁止盲目重发 |
| `SUCCESS_VISIBLE` | 1 | 已公开可见 | 永不自动重发 |
| `SUCCESS_PENDING_REVIEW` | 1 | 平台已接受，等待审核 | 永不自动重发 |
| `FAILED:<reason>` | 0 | 发布失败 | 临时失败可在后续轮次再试 |
| `SKIPPED:<reason>` | 0 | 规则/目标原因跳过 | 默认不重试 |
| `NEEDS_HUMAN:<reason>` | 0 | 本轮不能自动解决 | 本轮跳过；下轮可被动复核 |

## Critical unattended semantics

`NEEDS_HUMAN` 的含义不是“立刻暂停并叫用户处理”。

它表示：

> 该行在本轮无法自动完成，把阻塞持久化到 Excel，然后继续其他行。

单次运行期间不得为了以下情况停下来询问用户：

- 无法回答的入群问题；
- 单群验证码或弹窗；
- 单群页面异常；
- 单群禁止发布；
- 单群加入失败；
- 入群审核中。

只有当登录验证、账号限制、平台级 CAPTCHA、浏览器失效等导致**整个会话无法继续**时，才可以提前结束本轮；但仍必须先保存 Excel，并在最终结果中一次性说明。

## Recommended reason codes

### Membership

- `INVALID_TARGET`
- `UNREACHABLE_TARGET`
- `JOIN_NOT_AVAILABLE`
- `RULE_PROHIBITS`
- `QUESTION_REQUIRES_USER_FACT`
- `CAPTCHA`
- `LOGIN_REQUIRED`
- `ACCOUNT_RESTRICTED`
- `UNKNOWN`

### Sending

- `INVALID_TARGET`
- `UNREACHABLE_TARGET`
- `NOT_MEMBER`
- `POSTING_DISABLED`
- `RULE_PROHIBITS`
- `CONTENT_REJECTED`
- `CAPTCHA`
- `LOGIN_REQUIRED`
- `ACCOUNT_RESTRICTED`
- `SUBMIT_NOT_CONFIRMED`
- `UNKNOWN`

Examples:

```text
FAILED:UNREACHABLE_TARGET
NEEDS_HUMAN:QUESTION_REQUIRES_USER_FACT
NEEDS_HUMAN:CAPTCHA
SKIPPED:RULE_PROHIBITS
```

## Recovery from IN_PROGRESS

`IN_PROGRESS` is not safe-to-retry.

If a previous run ended with a row in `IN_PROGRESS`:

1. reopen the target;
2. verify current visible membership/post state;
3. only after verification choose a final state;
4. never blindly click Join/Post again.

For sending, duplicate prevention has higher priority than retry speed.

## Retry rules across iterations

### `PENDING_APPROVAL`

Every later run may re-check status. Do not submit another join request while the old request is still pending.

### `FAILED:*`

Transient failures may be retried on a later run. Permanent rule failures should be normalized to `SKIPPED:*`.

Because the schema intentionally remains seven columns, retry counters are not stored separately. Therefore the agent must use conservative retries: **at most one active retry per row per run**.

### `NEEDS_HUMAN:*`

A later run may passively check whether the blocker disappeared between runs. If it still requires user-specific facts, CAPTCHA, identity verification, or manual action, leave it unchanged and continue.

## Invariants

```text
已加入=1  => 入群状态 describes confirmed/successful membership
已发送=1  => 发送状态 is SUCCESS_VISIBLE or SUCCESS_PENDING_REVIEW
SUCCESS_VISIBLE => 已发送=1
SUCCESS_PENDING_REVIEW => 已发送=1
PENDING_APPROVAL => 已加入=0
FAILED:* => corresponding boolean stays 0 unless success is separately verified
NEEDS_HUMAN:* => corresponding boolean stays 0 unless success is separately verified
```

## User reset semantics

Between runs, the user may intentionally reset a row:

- retry membership from scratch: `已加入=0`, `入群状态=PENDING`;
- intentionally allow repost: `已发送=0`, `发送状态=PENDING`.

The agent must never perform these resets on its own merely to force another attempt.
