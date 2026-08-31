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
| E | 入群状态 | Text | Membership Lane 状态 |
| F | 已发送 | Integer Boolean | `0` or `1` |
| G | 发送状态 | Text | Posting Lane 状态 |

这七列既是输入，也是跨轮次持久化状态。不得依赖聊天记忆代替 Excel。

## Fundamental rule

Membership Lane 与 Posting Lane 独立：

```text
已加入=0 不能推出 不能发帖
已加入=1 不能推出 一定能发帖
```

因此以下组合合法：

```text
D=0 / E=PENDING_APPROVAL
F=1 / G=SUCCESS_VISIBLE
```

仅当平台明确要求 membership 时，Posting Lane 才依赖 Membership Lane。

## Membership state mapping

| 入群状态 | 已加入 | Meaning | Next-run behavior |
|---|---:|---|---|
| `PENDING` | 0 | 尚未处理 | 自动尝试 |
| `IN_PROGRESS` | 0 | 上轮可能中断 | 先验证真实状态再重试 |
| `SUCCESS` | 1 | 加入成功 | 不再入群 |
| `SUCCESS:ALREADY_MEMBER` | 1 | 页面确认原本已加入 | 不再入群 |
| `PENDING_APPROVAL` | 0 | 入群申请已提交 | 下轮自动复核，不重复申请 |
| `NOT_REQUIRED:POSTING_ALLOWED_WITHOUT_MEMBERSHIP` | 0 | 非成员已有发帖权限，成员身份不是当前发帖前置 | Posting Lane 可独立继续 |
| `FAILED:<reason>` | 0 | 入群失败 | 临时失败可后续再试 |
| `SKIPPED:<reason>` | 0 | 按规则主动跳过 | 默认不重试 |
| `NEEDS_HUMAN:<reason>` | 0 | 本轮无法自动解决 | 本轮跳过 Membership Lane，但仍检查 Posting Lane |

## Sending state mapping

| 发送状态 | 已发送 | Meaning | Next-run behavior |
|---|---:|---|---|
| `PENDING` | 0 | 尚未处理 | 自动尝试 |
| `IN_PROGRESS` | 0 | 上轮可能在提交附近中断 | 必须验证是否已经提交 |
| `SUCCESS_VISIBLE` | 1 | 已公开可见 | 永不自动重发 |
| `SUCCESS_PENDING_REVIEW` | 1 | 平台已接受，等待审核 | 永不自动重发 |
| `BLOCKED:MEMBERSHIP_REQUIRED` | 0 | 页面明确要求先成为成员 | 下轮先复核 Membership Lane |
| `FAILED:<reason>` | 0 | 发布失败 | 临时失败可后续再试 |
| `SKIPPED:<reason>` | 0 | 按规则跳过 | 默认不重试 |
| `NEEDS_HUMAN:<reason>` | 0 | 本轮不能自动解决 | 本轮跳过 Posting Lane |

## Join-question automation

入群问题由 Postman 在单轮中自动处理，答案来源优先级：

1. 用户本轮提供的事实；
2. 当前项目/业务固定事实；
3. `MEMBERSHIP_ANSWERS.md` 允许的安全模板。

所有必填问题都能真实回答时，自动填写并提交 Join 请求。

如果存在无法确定的事实型必填问题：

```text
D=0
E=NEEDS_HUMAN:QUESTION_REQUIRES_USER_FACT
```

该状态不得导致整批暂停，也不得阻止 Posting Lane 检查非成员发帖能力。

## Recommended reason codes

### Membership

- `INVALID_TARGET`
- `UNREACHABLE_TARGET`
- `JOIN_NOT_AVAILABLE`
- `RULE_PROHIBITS`
- `QUESTION_REQUIRES_USER_FACT`
- `QUESTION_FORM_SUBMIT_FAILED`
- `CAPTCHA`
- `LOGIN_REQUIRED`
- `ACCOUNT_RESTRICTED`
- `UNKNOWN`

### Sending

- `INVALID_TARGET`
- `UNREACHABLE_TARGET`
- `MEMBERSHIP_REQUIRED`
- `POSTING_DISABLED`
- `RULE_PROHIBITS`
- `CONTENT_REJECTED`
- `CAPTCHA`
- `LOGIN_REQUIRED`
- `ACCOUNT_RESTRICTED`
- `SUBMIT_NOT_CONFIRMED`
- `UNKNOWN`

## Critical unattended semantics

`NEEDS_HUMAN` 不表示立即暂停找用户。

它表示：该 Lane 本轮无法自动完成，把阻塞写入 Excel，继续同一行另一条 Lane，然后继续下一行。

单次运行期间不得因为以下情况停下来询问：

- 入群问题无法真实回答；
- 单群入群审核；
- 单群 Join 失败；
- 单群发帖失败；
- 单群页面异常。

只有登录验证、账号级限制、平台级 CAPTCHA、浏览器失效等导致整个会话无法继续时才提前结束；结束前必须保存 Excel。

## Recovery from IN_PROGRESS

`IN_PROGRESS` 不可盲目重试。

1. reopen target;
2. verify current visible membership/post state;
3. only then choose final state;
4. never blindly click Join/Post again.

Duplicate prevention has priority over retry speed.

## Retry rules

- `PENDING_APPROVAL`：下轮复核，不重复提交旧申请。
- `BLOCKED:MEMBERSHIP_REQUIRED`：成员状态变为1后可再次尝试发帖。
- `FAILED:*`：临时失败每轮最多主动重试一次。
- `NEEDS_HUMAN:*`：下轮可被动复核是否已解除。

## Invariants

```text
已加入=1 => 入群状态描述确认/成功成员资格
已发送=1 => 发送状态为 SUCCESS_VISIBLE 或 SUCCESS_PENDING_REVIEW
SUCCESS_VISIBLE => 已发送=1
SUCCESS_PENDING_REVIEW => 已发送=1
PENDING_APPROVAL => 已加入=0
NOT_REQUIRED:POSTING_ALLOWED_WITHOUT_MEMBERSHIP => 已加入=0
BLOCKED:MEMBERSHIP_REQUIRED => 已发送=0
NEEDS_HUMAN:* 不得自动推导另一条 Lane 失败
```

## User reset semantics

两轮之间用户可以主动重置：

- 重试入群：`已加入=0`, `入群状态=PENDING`；
- 明确允许重新发帖：`已发送=0`, `发送状态=PENDING`。

Agent 不得为了强行重试自行重置成功状态。
