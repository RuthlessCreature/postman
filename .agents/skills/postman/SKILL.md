---
name: postman
description: Generic unattended, resumable group/community posting workflow driven by a seven-column Excel queue. Membership and posting are independent state machines. Process all actionable rows without mid-run user interaction, check both lanes for every target, update Excel immediately after each side effect, defer row-level blockers, and resume from the updated workbook in the next iteration.
---

# Postman

## Purpose

Execute an **unattended, resumable dual-lane content-distribution workflow** across groups or communities using a user-provided Excel queue.

The user is not expected to intervene during a run. Process every currently actionable row, persist all results into the workbook, defer unresolved work, and return the updated workbook. The next run resumes from that workbook.

## Required inputs

1. One content payload to publish.
2. One `.xlsx` workbook containing exactly these seven columns in this order:
   - `序号`
   - `group名称`
   - `group url`
   - `已加入`
   - `入群状态`
   - `已发送`
   - `发送状态`

Do not silently rename, reorder, delete, or insert columns.

## Core model: TWO INDEPENDENT LANES

For every row, operate two logically independent state machines:

### Membership Lane

Controls only:

- `已加入`
- `入群状态`

### Posting Lane

Controls only:

- `已发送`
- `发送状态`

Never assume membership is required merely because `已加入=0`.

The following state is valid:

```text
已加入=0
入群状态=PENDING_APPROVAL
已发送=1
发送状态=SUCCESS_VISIBLE
```

It means the join request is still pending while the group allows non-members to publish.

The following is also valid:

```text
已加入=0
入群状态=NEEDS_HUMAN:QUESTION_REQUIRES_USER_FACT
已发送=1
发送状态=SUCCESS_PENDING_REVIEW
```

A blocker in Membership Lane MUST NOT suppress Posting Lane when posting is independently available.

Likewise, a posting failure MUST NOT prevent Membership Lane from advancing.

Only create a dependency when visible platform behavior explicitly requires membership before posting.

## Operating mode: UNATTENDED_BATCH

During one run:

- do not ask the user questions;
- do not request per-group approval;
- do not request posting confirmation;
- do not stop because one lane is blocked;
- do not stop because one row fails;
- do not stop after a few successes;
- process all rows that can be handled automatically;
- convert unresolved work into spreadsheet state and continue.

Human interaction, if needed, happens between runs.

A global hard blocker may end external actions early only when the entire browser/account session becomes unusable. Save the workbook first.

## Spreadsheet contract

The workbook is the Single Source of Truth.

### Membership Lane

`已加入`:

- `1`: confirmed member.
- `0`: not confirmed as a member.

| Outcome | 已加入 | 入群状态 |
|---|---:|---|
| Already a member | 1 | `SUCCESS:ALREADY_MEMBER` |
| Join succeeds | 1 | `SUCCESS` |
| Join request pending | 0 | `PENDING_APPROVAL` |
| Posting works without membership and joining is unnecessary/unavailable | 0 | `NOT_REQUIRED:POSTING_ALLOWED_WITHOUT_MEMBERSHIP` |
| Join fails | 0 | `FAILED:<reason>` |
| Manual fact/action required | 0 | `NEEDS_HUMAN:<reason>` |
| Join intentionally skipped | 0 | `SKIPPED:<reason>` |

`NOT_REQUIRED:*` never changes `已加入` to 1.

### Posting Lane

`已发送`:

- `1`: platform accepted this content submission.
- `0`: submission is not confirmed accepted.

| Outcome | 已发送 | 发送状态 |
|---|---:|---|
| Post visible | 1 | `SUCCESS_VISIBLE` |
| Accepted for moderation | 1 | `SUCCESS_PENDING_REVIEW` |
| Membership explicitly required before posting | 0 | `BLOCKED:MEMBERSHIP_REQUIRED` |
| Submission failed | 0 | `FAILED:<reason>` |
| Manual action required | 0 | `NEEDS_HUMAN:<reason>` |
| Posting intentionally skipped | 0 | `SKIPPED:<reason>` |

`SUCCESS_PENDING_REVIEW` MUST set `已发送=1` to prevent duplicate submission.

## Start-of-run recovery sweep

Process rows in ascending `序号` unless specified otherwise.

### Posting recovery

- `已发送=1` → never publish again unless the user reset it between runs.
- `发送状态=IN_PROGRESS` → verify actual page/post state before any retry.
- `BLOCKED:MEMBERSHIP_REQUIRED` → re-check Membership Lane. If membership is now confirmed, Posting Lane becomes actionable.
- `FAILED:*` → retry only if plausibly transient, at most once per run.
- `NEEDS_HUMAN:*` → passively re-check once if external state may have changed; otherwise keep and continue.

### Membership recovery

- `已加入=1` → do not rejoin.
- `入群状态=IN_PROGRESS` → verify real state before retry.
- `PENDING_APPROVAL` → re-check whether approval happened; do not blindly resubmit.
- `FAILED:*` → retry only when transient.
- `NEEDS_HUMAN:*` → passively re-check once; never fabricate unknown facts.

## Per-row algorithm

### Step 1 — Open once and inspect both lanes

Open the target and determine from visible evidence:

- current membership state;
- whether normal joining is available;
- whether a composer/posting entry is already available to the current account;
- whether the page explicitly says membership is required to post;
- whether group/community rules prohibit the content.

Do not infer posting permission only from membership state.

### Step 2 — Advance Membership Lane independently

If `已加入=0`:

1. If already a member → `1 / SUCCESS:ALREADY_MEMBER`.
2. Else if a normal join flow is available → attempt once.
3. Answer only membership questions whose answers are explicitly known from user-provided facts.
4. Unknown user-specific fact → `0 / NEEDS_HUMAN:QUESTION_REQUIRES_USER_FACT`.
5. Immediate join success → `1 / SUCCESS`.
6. Join request submitted → `0 / PENDING_APPROVAL`.
7. Join failure → `0 / FAILED:<reason>`.
8. If joining is unnecessary/unavailable while posting is already permitted → `0 / NOT_REQUIRED:POSTING_ALLOWED_WITHOUT_MEMBERSHIP` when this accurately describes the page.
9. Checkpoint after each material change.

Do NOT `continue` to the next row merely because membership remains 0. Posting Lane still must be evaluated.

### Step 3 — Advance Posting Lane independently

If `已发送=0`:

1. If a posting composer/action is currently available and rules permit the content, attempt publication even when `已加入=0`.
2. If the platform explicitly states membership is required and no posting path is currently available → `0 / BLOCKED:MEMBERSHIP_REQUIRED`.
3. Before an actual submission, set `发送状态=IN_PROGRESS` and checkpoint if practical.
4. Insert the user-provided content without inventing or altering material facts.
5. Submit once.
6. Verify visible outcome.
7. Visible → `1 / SUCCESS_VISIBLE`.
8. Accepted for review → `1 / SUCCESS_PENDING_REVIEW`.
9. Failed → `0 / FAILED:<reason>`.
10. Manual requirement → `0 / NEEDS_HUMAN:<reason>`.
11. Checkpoint and continue.

### Step 4 — Reconcile explicit dependency only when real

If Posting Lane is `BLOCKED:MEMBERSHIP_REQUIRED` and Membership Lane becomes `已加入=1` in the same visit, immediately re-check whether the posting composer is now available. If yes, proceed to publish in the same run.

If membership remains pending but posting was independently available and succeeded, preserve both independent states. Do not force Membership Lane to success.

## Valid combinations

These are valid and must not be normalized away:

```text
D=0 / E=PENDING_APPROVAL
F=1 / G=SUCCESS_VISIBLE
```

```text
D=0 / E=FAILED:JOIN_NOT_AVAILABLE
F=1 / G=SUCCESS_PENDING_REVIEW
```

```text
D=1 / E=SUCCESS
F=0 / G=FAILED:POSTING_DISABLED
```

```text
D=0 / E=PENDING_APPROVAL
F=0 / G=BLOCKED:MEMBERSHIP_REQUIRED
```

## Invalid assumptions

Never assume:

```text
已加入=0 => 不能发
已加入=1 => 一定能发
已发送=1 => 已加入=1
入群失败 => 发帖失败
发帖失败 => 入群失败
```

## Row-level blockers versus global hard blockers

Row-level problems MUST NOT terminate the batch:

- group unavailable;
- join pending/denied;
- membership question needs unknown facts;
- membership unavailable;
- posting disabled for one group;
- content prohibited by one group;
- one lane fails while the other can proceed;
- one page structure is unusable.

Global blockers may stop external actions only when they make the whole session unusable:

- session-wide login/device verification;
- CAPTCHA blocking all further navigation/actions;
- account-wide posting restriction;
- browser/computer-use capability unavailable.

Before stopping: update the current row when relevant, checkpoint the workbook, do not ask the user mid-run, and return the updated workbook.

## Checkpoint discipline

Checkpoint after every material state transition in either lane. Never wait until the whole batch finishes.

If the environment only permits writing the file at the end, maintain an exact mutation log and apply every completed mutation before returning. Never claim a workbook update unless the updated file actually exists.

## Idempotency

1. `已发送=1` is an absolute duplicate-post guard.
2. `已加入=1` is an absolute duplicate-join guard.
3. `SUCCESS_PENDING_REVIEW` counts as sent.
4. `PENDING_APPROVAL` is re-checked later, not blindly re-submitted.
5. `IN_PROGRESS` in either lane must be verified before retry.
6. `BLOCKED:MEMBERSHIP_REQUIRED` becomes retryable when Membership Lane later reaches 1.
7. A Membership Lane blocker never suppresses an independently available Posting Lane.
8. A Posting Lane blocker never suppresses Membership Lane.

## Compliance

Do not bypass CAPTCHA, access controls, login challenges, group rules, administrator approval, platform restrictions, rate limits, or anti-spam protections. Do not use deceptive identities, account rotation, proxy rotation, or fingerprint evasion.

## End-of-run condition

Continue until:

- every row has been scanned and both lanes evaluated where applicable;
- all remaining lane actions are non-actionable this run;
- a global hard blocker makes the current session unusable;
- a user-defined batch limit is reached.

Do not stop merely because some rows were already completed.

## Final report

Return once at the end:

- updated workbook;
- rows inspected;
- membership actions attempted/succeeded/pending/failed/deferred;
- posting actions attempted/visible/pending-review/blocked/failed/deferred;
- rows where posting succeeded without membership;
- unresolved row numbers for next run;
- global blocker if any.

Do not interrupt the run with user-facing questions.
