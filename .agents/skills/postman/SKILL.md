---
name: postman
description: Generic resumable group/community posting workflow driven by a seven-column Excel queue. Use when the user provides content plus a spreadsheet containing group name/URL, membership and sending status. The spreadsheet is the single source of truth. Process rows incrementally, update status immediately after each side effect, skip completed rows, and continue after row-level failures.
---

# Postman

## Purpose

Execute a resumable content-distribution workflow across groups or communities using a user-provided Excel queue.

This skill is platform-agnostic. It may be used with Facebook groups or other community platforms only when the current environment has an authorized browser/computer-use capability and the requested actions comply with platform/group rules.

## Required inputs

1. One content payload to publish. It may be:
   - pasted in chat;
   - provided as a text/Markdown file;
   - explicitly referenced by the user.
2. One `.xlsx` workbook containing exactly these seven columns in this order:
   - `序号`
   - `group名称`
   - `group url`
   - `已加入`
   - `入群状态`
   - `已发送`
   - `发送状态`

Do not silently rename, reorder, delete, or insert columns into the user's seven-column queue.

## Spreadsheet contract

The workbook is the Single Source of Truth.

### `已加入`

Boolean integer only:

- `1`: membership/access required for posting has been confirmed.
- `0`: not confirmed.

Membership outcomes:

| Outcome | 已加入 | 入群状态 |
|---|---:|---|
| Already a member | 1 | `SUCCESS:ALREADY_MEMBER` |
| Join succeeds immediately | 1 | `SUCCESS` |
| Join request submitted, pending approval | 0 | `PENDING_APPROVAL` |
| Join fails | 0 | `FAILED:<reason>` |
| Human input required | 0 | `NEEDS_HUMAN:<reason>` |
| Not eligible / rule prevents joining | 0 | `SKIPPED:<reason>` |

### `已发送`

Boolean integer only:

- `1`: the platform has accepted the content submission.
- `0`: submission has not been accepted.

Sending outcomes:

| Outcome | 已发送 | 发送状态 |
|---|---:|---|
| Post visibly published | 1 | `SUCCESS_VISIBLE` |
| Post accepted and waiting for moderator/admin review | 1 | `SUCCESS_PENDING_REVIEW` |
| Submission failed | 0 | `FAILED:<reason>` |
| Human input required | 0 | `NEEDS_HUMAN:<reason>` |
| Posting not allowed / intentionally skipped | 0 | `SKIPPED:<reason>` |

Important: `SUCCESS_PENDING_REVIEW` MUST set `已发送=1`. Otherwise the next run could duplicate the post while the first submission is still in moderation.

## State machine

Process rows in ascending `序号` unless the user specifies another ordering.

For each row:

### Step 1 — Skip completed work

If `已发送 == 1`:

- do not post again;
- do not modify the row unless the user explicitly asks to re-verify it;
- continue to the next row.

### Step 2 — Validate target

Require:

- non-empty `group名称`;
- syntactically usable `group url`;
- target page can be opened or otherwise resolved.

If target is invalid:

- leave `已加入` unchanged unless evidence proves membership;
- set the relevant status to `FAILED:INVALID_TARGET` or `FAILED:UNREACHABLE_TARGET`;
- save the workbook;
- continue.

### Step 3 — Resolve membership

If `已加入 == 0`:

1. Open the target group/community.
2. Determine current membership/access state from visible evidence.
3. If already a member:
   - set `已加入=1`;
   - set `入群状态=SUCCESS:ALREADY_MEMBER`;
   - immediately save/checkpoint the workbook.
4. If joining is required and permitted:
   - attempt the normal join flow;
   - answer only questions whose answers are explicitly known from user-provided facts;
   - never invent identity, employment, location, affiliation, or personal-history answers.
5. Update the row immediately:
   - immediate membership success → `1 / SUCCESS`;
   - request submitted → `0 / PENDING_APPROVAL`;
   - failure → `0 / FAILED:<reason>`;
   - human requirement → `0 / NEEDS_HUMAN:<reason>`.
6. Save/checkpoint immediately.

If membership remains `0`, do not attempt posting for that row in the same pass unless the platform explicitly allows posting without membership.

### Step 4 — Publish

Only attempt publication when:

- `已加入 == 1`, or posting is explicitly available without membership;
- `已发送 == 0`;
- group/community rules do not prohibit the requested content;
- there is no account-level blocker.

Before the side effect:

- set `发送状态=IN_PROGRESS`;
- save/checkpoint if practical.

Then:

1. Open the composer.
2. Insert the exact user-approved content.
3. Make only transformations explicitly allowed by the user.
4. Submit once.
5. Verify visible outcome.

Update immediately:

- visible post → `已发送=1`, `发送状态=SUCCESS_VISIBLE`;
- accepted for moderation → `已发送=1`, `发送状态=SUCCESS_PENDING_REVIEW`;
- failure → `已发送=0`, `发送状态=FAILED:<reason>`;
- human intervention → `已发送=0`, `发送状态=NEEDS_HUMAN:<reason>`.

Save/checkpoint immediately after the result is known.

## Loop behavior

A row-level failure MUST NOT end the batch.

After updating and saving the failed row, continue to the next eligible row.

Examples of row-level failures:

- group unavailable;
- membership pending;
- join denied;
- posting disabled for one group;
- group rule prohibits the content;
- page structure for one group is unusable.

Account/session-level blockers MAY stop side effects across all rows:

- CAPTCHA;
- login/device verification;
- account restriction;
- platform-wide posting restriction;
- browser/computer-use capability unavailable.

When an account-level blocker occurs:

1. update the current row with `NEEDS_HUMAN:<reason>` when relevant;
2. save the workbook;
3. stop further external side effects;
4. report exactly what the user must do before resuming.

## Checkpoint discipline

Never wait until the end of the entire batch to save state.

Checkpoint after every material state transition, especially after:

- confirmed existing membership;
- join request submission;
- join failure;
- successful join;
- post submission;
- post failure;
- moderation-pending result;
- human-blocker detection.

If the environment only permits returning a modified file at the end of a turn, keep an in-memory mutation log and apply/save every completed row before returning. Never claim that a workbook was updated unless an updated file actually exists.

## Idempotency rules

1. `已发送=1` is an absolute duplicate-post guard.
2. `PENDING_APPROVAL` rows are not re-joined unless visible evidence shows the prior request is gone/rejected or the user explicitly requests retry.
3. `SUCCESS_PENDING_REVIEW` is treated as already sent.
4. `FAILED:*` may be retried in a later run unless the reason is a permanent rule/policy prohibition.
5. `SKIPPED:*` is not retried unless the user explicitly resets/reclassifies it.
6. `NEEDS_HUMAN:*` is not repeatedly hammered in the same run.

## Content integrity

Default behavior is to publish the content exactly as provided.

Do not:

- invent salaries, benefits, employers, locations, qualifications, or contact details;
- alter links or phone numbers;
- add claims to improve conversion;
- generate deceptive answers to membership questions.

If the user authorizes localized introductions or platform-specific formatting, preserve all substantive facts.

## Compliance and platform behavior

Do not bypass:

- CAPTCHA;
- access controls;
- login challenges;
- group rules;
- administrator approval;
- platform restrictions;
- rate limits or anti-spam protections.

Do not use account rotation, stealth/fingerprint evasion, proxy rotation, or deceptive identity claims to avoid enforcement.

## Final report

At the end of each run, report:

- rows inspected;
- rows skipped because `已发送=1`;
- newly joined;
- join requests pending;
- join failures;
- posts visible;
- posts pending review;
- post failures;
- human blockers;
- first next eligible `序号` for the next run.

The updated workbook is the authoritative deliverable.
