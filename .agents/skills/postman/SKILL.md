---
name: postman
description: Generic unattended, resumable group/community posting workflow driven by a seven-column Excel queue. Use when the user provides content plus a spreadsheet containing group name/URL, membership and sending status. The spreadsheet is the single source of truth. Process all currently actionable rows without mid-run user interaction, update status immediately after each side effect, skip completed rows, defer row-level blockers, and resume from the updated workbook in the next iteration.
---

# Postman

## Purpose

Execute an **unattended batch** content-distribution workflow across groups or communities using a user-provided Excel queue.

The user is not expected to intervene during a run. A run should process every currently actionable row, persist results into the workbook, defer unresolved rows, and return the updated workbook. The next run resumes from that workbook.

This skill is platform-agnostic. It may be used with Facebook groups or other community platforms only when the environment has an authorized browser/computer-use capability and the requested actions comply with platform/group rules.

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

## Core operating mode: UNATTENDED_BATCH

During one run:

- do not ask the user questions;
- do not request per-group approval;
- do not request posting confirmation;
- do not wait for the user to solve row-level blockers;
- do not stop after completing only a few rows;
- process all rows that can be handled automatically;
- convert unresolved work into spreadsheet state and continue.

Human interaction, if needed, happens **between runs**, not during a run.

The only reason to end external actions early is a **global hard blocker** that makes the entire current browser/account session unusable. Even then, save the workbook first and return it.

## Spreadsheet contract

The workbook is the Single Source of Truth.

### `已加入`

- `1`: membership/access required for posting has been confirmed.
- `0`: not confirmed.

| Outcome | 已加入 | 入群状态 |
|---|---:|---|
| Already a member | 1 | `SUCCESS:ALREADY_MEMBER` |
| Join succeeds immediately | 1 | `SUCCESS` |
| Join request submitted, pending approval | 0 | `PENDING_APPROVAL` |
| Join fails | 0 | `FAILED:<reason>` |
| Requires action outside this run | 0 | `NEEDS_HUMAN:<reason>` |
| Not eligible / rule prevents joining | 0 | `SKIPPED:<reason>` |

### `已发送`

- `1`: the platform has accepted the content submission.
- `0`: submission has not been accepted or confirmed.

| Outcome | 已发送 | 发送状态 |
|---|---:|---|
| Post visibly published | 1 | `SUCCESS_VISIBLE` |
| Accepted and waiting for moderator/admin review | 1 | `SUCCESS_PENDING_REVIEW` |
| Submission failed | 0 | `FAILED:<reason>` |
| Requires action outside this run | 0 | `NEEDS_HUMAN:<reason>` |
| Posting intentionally skipped | 0 | `SKIPPED:<reason>` |

`SUCCESS_PENDING_REVIEW` MUST set `已发送=1` to prevent duplicate submission on the next run.

## Start-of-run recovery sweep

Process rows in ascending `序号` unless the user specifies another order.

Before normal processing, interpret unresolved states:

### `已发送=1`

Never publish again unless the user explicitly reset it between runs.

### `IN_PROGRESS`

Treat as uncertain. Re-open the target and verify visible membership/post state before retrying any side effect. Never blindly click Join/Post again.

### `PENDING_APPROVAL`

Re-check membership:

- approved → set `已加入=1`, then continue to posting;
- still pending → keep state and continue to next row;
- rejected/request disappeared → record current evidence and, if normal joining is still available, one fresh join attempt may be made.

### `FAILED:*`

- transient failures may be retried once in a later run;
- permanent rule/policy failures should be `SKIPPED:*`;
- do not hammer the same failing action repeatedly in one run.

### `NEEDS_HUMAN:*`

This does **not** mean pause and ask the user now.

- passively re-check once if the external state could have changed between runs;
- if the same manual requirement remains, keep the state and continue;
- if resolution requires unknown user facts, CAPTCHA, identity verification, or other manual action, do not fabricate or bypass it.

## Per-row state machine

### Step 1 — Skip completed work

If `已发送 == 1`, continue immediately to the next row.

### Step 2 — Validate target

Require usable `group名称` and `group url`.

If invalid/unreachable:

- record `FAILED:INVALID_TARGET` or `FAILED:UNREACHABLE_TARGET`;
- checkpoint;
- continue.

### Step 3 — Resolve membership

If `已加入 == 0`:

1. Open the target.
2. Determine current membership/access from visible evidence.
3. Already a member → `1 / SUCCESS:ALREADY_MEMBER` and checkpoint.
4. If normal joining is permitted, attempt it once.
5. Answer only questions whose answers are explicitly known from user-provided facts.
6. If a question requires unknown identity, location, employment, affiliation, invitation, or personal-history facts → `0 / NEEDS_HUMAN:QUESTION_REQUIRES_USER_FACT`, checkpoint, continue to next row.
7. Immediate success → `1 / SUCCESS`.
8. Request submitted → `0 / PENDING_APPROVAL`.
9. Failure → `0 / FAILED:<reason>`.
10. Checkpoint after each material transition.

If membership remains `0` and posting requires membership, continue to the next row.

### Step 4 — Publish

Only attempt when:

- `已发送 == 0`;
- posting permission is available;
- group/community rules do not prohibit the content;
- no global account-level blocker exists.

Before submission, set `发送状态=IN_PROGRESS` and checkpoint if practical.

Then:

1. Open composer.
2. Insert the user-provided content.
3. Preserve substantive facts and contact details.
4. Submit once.
5. Verify visible outcome.
6. Visible post → `已发送=1 / SUCCESS_VISIBLE`.
7. Accepted for moderation → `已发送=1 / SUCCESS_PENDING_REVIEW`.
8. Failure → `已发送=0 / FAILED:<reason>`.
9. Manual requirement → `已发送=0 / NEEDS_HUMAN:<reason>`.
10. Checkpoint and continue.

## Row-level blockers versus global hard blockers

### Row-level blockers — MUST continue batch

Examples:

- group unavailable;
- membership pending;
- join denied;
- membership question requires unknown user fact;
- posting disabled for one group;
- group rule prohibits the content;
- one page structure is unusable;
- one action times out or fails.

Response: update that row, save, continue.

### Global hard blockers — may end external actions

Only when the blocker makes the whole current browser/account session unusable:

- session-wide login/device verification;
- CAPTCHA that blocks all further navigation/actions;
- account-level/platform-wide posting restriction;
- browser/computer-use capability unavailable.

Response:

1. update current row when relevant;
2. checkpoint the workbook;
3. do not ask the user mid-run;
4. stop further external side effects;
5. return the updated workbook and blocker summary.

If a CAPTCHA or error affects only one row and the session remains usable elsewhere, treat it as row-level and continue.

## Checkpoint discipline

Checkpoint after every material state transition, especially after:

- membership confirmation;
- join submission;
- join failure;
- successful join;
- post submission;
- post failure;
- moderation-pending result;
- deferred manual blocker.

If the environment only permits returning a modified file at the end of a turn, keep an exact mutation log and apply/save all completed row changes before returning. Never claim an update unless the updated workbook actually exists.

## Idempotency rules

1. `已发送=1` is an absolute duplicate-post guard.
2. `PENDING_APPROVAL` is re-checked later, not blindly re-submitted.
3. `SUCCESS_PENDING_REVIEW` counts as already sent.
4. `IN_PROGRESS` must be verified before retry.
5. `FAILED:*` may be retried on a later run only when the failure is plausibly transient.
6. `SKIPPED:*` is not retried unless reset/reclassified.
7. `NEEDS_HUMAN:*` is never hammered repeatedly in the same run.
8. A row-level failure never terminates the batch.

## Content integrity

Default behavior is to publish the content exactly as provided.

Do not invent salaries, benefits, employers, locations, qualifications, contact details, or membership-question answers. If localized formatting is authorized, preserve all substantive facts.

## Compliance

Do not bypass CAPTCHA, access controls, login challenges, group rules, administrator approval, platform restrictions, rate limits, or anti-spam protections. Do not use account rotation, stealth/fingerprint evasion, proxy rotation, or deceptive identities.

## End-of-run condition

Continue until one of these is true:

- every row has been scanned;
- all remaining rows are non-actionable this run;
- a global hard blocker makes the current session unusable;
- a user-defined batch limit has been reached.

Do not stop merely because some work has already been completed.

## Final report

Return once, at the end of the run:

- updated workbook;
- rows inspected;
- completed rows skipped;
- newly joined;
- join requests pending;
- join failures;
- deferred `NEEDS_HUMAN` rows;
- posts visible;
- posts pending review;
- post failures;
- skipped rows;
- unresolved row numbers for the next run;
- global blocker, if any.

Do not interrupt the run with user-facing questions.
