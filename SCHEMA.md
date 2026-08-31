# Postman Excel Schema

## Sheet

默认工作表名建议为 `Groups`。如果用户上传的现有文件使用其他工作表名，只要包含完整七列即可，不强制改名。

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

## Membership state mapping

| 入群状态 | 已加入 | Meaning | Retry default |
|---|---:|---|---|
| `PENDING` | 0 | 尚未处理 | Yes |
| `IN_PROGRESS` | 0 | 当前正在处理 | Resume/check |
| `SUCCESS` | 1 | 本轮加入成功 | No |
| `SUCCESS:ALREADY_MEMBER` | 1 | 页面确认原本已加入 | No |
| `PENDING_APPROVAL` | 0 | 入群申请已提交，等待审批 | Recheck later, do not re-submit blindly |
| `FAILED:<reason>` | 0 | 入群失败 | Usually yes in a later run |
| `SKIPPED:<reason>` | 0 | 按规则主动跳过 | No unless reset |
| `NEEDS_HUMAN:<reason>` | 0 | 需要人工处理 | No until human action |

## Sending state mapping

| 发送状态 | 已发送 | Meaning | Retry default |
|---|---:|---|---|
| `PENDING` | 0 | 尚未处理 | Yes |
| `IN_PROGRESS` | 0 | 正在执行发布 | Verify before any retry |
| `SUCCESS_VISIBLE` | 1 | 已公开可见 | Never automatically retry |
| `SUCCESS_PENDING_REVIEW` | 1 | 平台已接受提交，等待审核 | Never automatically retry |
| `FAILED:<reason>` | 0 | 发布失败 | Usually yes in a later run |
| `SKIPPED:<reason>` | 0 | 按规则主动跳过 | No unless reset |
| `NEEDS_HUMAN:<reason>` | 0 | 需要人工处理 | No until human action |

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
SKIPPED:RULE_PROHIBITS
```

## Recovery from IN_PROGRESS

`IN_PROGRESS` is deliberately not treated as safe-to-retry.

If a previous run crashed while a row is `IN_PROGRESS`:

1. reopen the target;
2. verify current visible membership/post state;
3. only after verification choose the correct final state;
4. do not blindly click Join/Post again.

For sending in particular, if there is any evidence the post may already have been submitted, treat duplicate prevention as higher priority than retry speed.

## Invariants

The following must always be true:

```text
已加入=1  => 入群状态 must describe successful/confirmed membership
已发送=1  => 发送状态 must be SUCCESS_VISIBLE or SUCCESS_PENDING_REVIEW
SUCCESS_VISIBLE => 已发送=1
SUCCESS_PENDING_REVIEW => 已发送=1
PENDING_APPROVAL => 已加入=0
FAILED:* => corresponding boolean remains 0 unless success was separately verified
```

## User reset semantics

The user may intentionally reset a row to make it eligible again.

Examples:

- retry membership: set `已加入=0`, `入群状态=PENDING`;
- intentionally repost: set `已发送=0`, `发送状态=PENDING`.

The agent must never perform these resets on its own merely to retry an action.
