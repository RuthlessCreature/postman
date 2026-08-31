---
name: postman
description: Generic unattended, resumable group/community posting workflow driven by a seven-column Excel queue. Membership and posting are independent state machines. Automatically answer and submit join questions only when answers are grounded in runtime-supplied facts or generic approved answer templates. Process all actionable rows without mid-run user interaction, checkpoint Excel after each side effect, defer unresolved row-level blockers, and resume from the updated workbook next iteration.
---

# Postman

## Purpose

Run an **unattended, resumable, business-agnostic dual-lane content-distribution workflow** across groups or communities using a user-provided Excel queue.

Postman MUST NOT assume any specific company, brand, industry, campaign, geography, job type, product, identity, or posting purpose. All task-specific facts come from runtime input.

The user does not intervene during a run. Process every currently actionable row, persist all results into the workbook, defer unresolved work, and return the updated workbook. The next run resumes from that workbook.

Before execution, read `MEMBERSHIP_ANSWERS.md` when available.

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
3. Optional runtime facts/configuration supplied by the user for this specific run, such as name, organization, role, location, website, contact, purpose, content type, or other known facts.

Do not rename, reorder, delete, or insert Excel columns.

Never invent a missing runtime fact.

## Core model: TWO INDEPENDENT LANES

For every row operate two independent state machines.

### Membership Lane

Controls only `已加入` and `入群状态`.

### Posting Lane

Controls only `已发送` and `发送状态`.

Never assume membership is required because `已加入=0`.

Valid example:

```text
已加入=0
入群状态=PENDING_APPROVAL
已发送=1
发送状态=SUCCESS_VISIBLE
```

A Membership Lane blocker MUST NOT suppress Posting Lane when posting is independently available. A Posting Lane blocker MUST NOT prevent Membership Lane from advancing.

Only create a dependency when the platform visibly requires membership before posting.

## Operating mode: UNATTENDED_BATCH

During one run:

- do not ask the user questions;
- do not request per-group approval;
- do not request posting confirmation;
- do not pause for join questions;
- do not stop because one lane or row is blocked;
- do not stop after a few successes;
- process all automatically actionable rows;
- convert unresolved work into spreadsheet state and continue.

Human interaction, if ever needed, happens between runs.

## Spreadsheet contract

The workbook is the Single Source of Truth.

### Membership Lane states

| Outcome | 已加入 | 入群状态 |
|---|---:|---|
| Already member | 1 | `SUCCESS:ALREADY_MEMBER` |
| Join succeeds | 1 | `SUCCESS` |
| Join request submitted | 0 | `PENDING_APPROVAL` |
| Posting allowed without membership and joining unnecessary/unavailable | 0 | `NOT_REQUIRED:POSTING_ALLOWED_WITHOUT_MEMBERSHIP` |
| Join fails | 0 | `FAILED:<reason>` |
| Unknown fact/manual action required | 0 | `NEEDS_HUMAN:<reason>` |
| Join intentionally skipped | 0 | `SKIPPED:<reason>` |

### Posting Lane states

| Outcome | 已发送 | 发送状态 |
|---|---:|---|
| Post visible | 1 | `SUCCESS_VISIBLE` |
| Accepted for moderation | 1 | `SUCCESS_PENDING_REVIEW` |
| Membership explicitly required before posting | 0 | `BLOCKED:MEMBERSHIP_REQUIRED` |
| Submission failed | 0 | `FAILED:<reason>` |
| Manual action required | 0 | `NEEDS_HUMAN:<reason>` |
| Posting intentionally skipped | 0 | `SKIPPED:<reason>` |

`SUCCESS_PENDING_REVIEW` MUST set `已发送=1`.

## Start-of-run recovery

- `已发送=1` → never publish again unless user reset it between runs.
- `已加入=1` → never join again.
- `IN_PROGRESS` → verify real state before retry.
- `PENDING_APPROVAL` → re-check approval; do not blindly resubmit.
- `BLOCKED:MEMBERSHIP_REQUIRED` → re-check Membership Lane; retry posting only if membership is now confirmed.
- transient `FAILED:*` → at most one retry per run.
- `NEEDS_HUMAN:*` → passively re-check once; if still unresolved, keep state and continue.

## Per-row algorithm

### Step 1 — Open once, inspect both lanes

Determine from visible evidence:

- current membership state;
- whether normal joining is available;
- whether join questions are present;
- whether posting/composer is currently available even without membership;
- whether membership is explicitly required before posting;
- whether group/community rules prohibit the supplied content.

Do not infer posting permission only from membership state.

### Step 2 — Advance Membership Lane

If `已加入=0`:

1. Already member → `1 / SUCCESS:ALREADY_MEMBER`.
2. Else if Join is available, start the normal join flow once.
3. If join questions appear, run **Join Question Auto-Fill** below.
4. If all required questions can be answered automatically, fill them and submit the join request without asking the user.
5. Immediate join success → `1 / SUCCESS`.
6. Request accepted for admin review → `0 / PENDING_APPROVAL`.
7. Failure → `0 / FAILED:<reason>`.
8. If joining is unnecessary/unavailable while posting is already permitted → `0 / NOT_REQUIRED:POSTING_ALLOWED_WITHOUT_MEMBERSHIP` when accurate.
9. Checkpoint after each material change.

Do NOT continue to the next row merely because membership remains 0. Posting Lane still must be evaluated.

## Join Question Auto-Fill

When membership questions appear, answer them automatically using this precedence:

1. facts explicitly supplied in the current task;
2. optional runtime identity/business/context facts supplied with this run;
3. `MEMBERSHIP_ANSWERS.md` generic approved templates.

The Skill itself has no built-in organization, role, business, industry, location, or campaign identity.

### Automatically answer generic rule questions

Examples:

- `Do you agree to the group rules?` → `Yes.` only after the rules were actually checked.
- `Will you avoid spam / irrelevant posts?` → `Yes.`
- `Will you follow posting guidelines?` → `Yes.`

### Purpose / intent questions

For questions such as `Why do you want to join?`, generate a short truthful response from runtime-supplied purpose/content facts.

Generic form when facts support it:

```text
I would like to join this community to participate in relevant discussions and share content related to <known purpose/content type> where permitted. I will follow the group rules.
```

If no specific purpose is supplied and the question accepts a generic community-participation answer, use:

```text
I would like to join the community and participate in relevant discussions while following the group rules.
```

If the question explicitly requires a specific identity, affiliation, location, occupation, or other fact that is unknown, do not guess.

### Automatically select checkboxes/multiple-choice answers

Select an option only when it truthfully matches runtime-supplied facts.

Examples:

- agree to rules;
- user/customer/member type explicitly supplied in runtime facts;
- business/recruiter/student/teacher/creator/etc. only if explicitly known.

Do not choose a city, nationality, employer, organization, profession, relationship, or identity merely to pass admission.

### Unknown factual question

If a required question asks for an unknown fact such as current residence, nationality, employer, organization, inviter/member name, personal history, verification data, or any other identity-specific information:

```text
已加入=0
入群状态=NEEDS_HUMAN:QUESTION_REQUIRES_USER_FACT
```

Do not fabricate. Do not pause to ask the user. Save the row and continue evaluating Posting Lane.

### Submit behavior

When all required join questions are answerable:

1. fill every required field;
2. tick required rule acknowledgements only when truthful;
3. submit `Join Group` / `Submit` once;
4. verify visible outcome;
5. write `SUCCESS`, `PENDING_APPROVAL`, or `FAILED:<reason>`;
6. checkpoint immediately;
7. continue Posting Lane regardless of join outcome.

## Step 3 — Advance Posting Lane

If `已发送=0`:

1. If composer/posting is currently available and rules permit the supplied content, publish even when `已加入=0`.
2. If page explicitly requires membership and no posting path is available → `0 / BLOCKED:MEMBERSHIP_REQUIRED`.
3. Before actual submission set `发送状态=IN_PROGRESS` and checkpoint if practical.
4. Insert the user-provided content without inventing or changing material facts.
5. Submit once.
6. Verify outcome.
7. Visible → `1 / SUCCESS_VISIBLE`.
8. Accepted for review → `1 / SUCCESS_PENDING_REVIEW`.
9. Failed → `0 / FAILED:<reason>`.
10. Manual requirement → `0 / NEEDS_HUMAN:<reason>`.
11. Checkpoint and continue.

## Step 4 — Reconcile explicit dependency only when real

If Posting Lane is `BLOCKED:MEMBERSHIP_REQUIRED` and Membership Lane reaches `已加入=1` in the same visit, re-check posting immediately and publish if now available.

If membership remains pending but posting independently succeeds, preserve both independent states.

## Valid combinations

```text
D=0 / E=PENDING_APPROVAL
F=1 / G=SUCCESS_VISIBLE
```

```text
D=0 / E=NEEDS_HUMAN:QUESTION_REQUIRES_USER_FACT
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
有入群问题 => 必须停下来问用户
帖子主题 => 用户身份/职业/组织关系
```

## Row-level vs global blockers

Row-level problems MUST NOT terminate the batch:

- group unavailable;
- join pending/denied;
- one required join answer is unknown;
- membership unavailable;
- posting disabled for one group;
- content prohibited by one group;
- one lane fails while the other can proceed;
- one page structure is unusable.

Global blockers may stop external actions only when the whole session becomes unusable:

- session-wide login/device verification;
- CAPTCHA blocking all further navigation/actions;
- account-wide posting restriction;
- browser/computer-use capability unavailable.

Before stopping: update current state, checkpoint workbook, do not ask mid-run, return the updated workbook.

## Checkpoint discipline

Checkpoint after every material state transition in either lane, including join-question submission and join-request result.

If the environment only permits writing at end of turn, maintain an exact mutation log and apply every mutation before returning.

## Idempotency

1. `已发送=1` is absolute duplicate-post guard.
2. `已加入=1` is absolute duplicate-join guard.
3. `SUCCESS_PENDING_REVIEW` counts as sent.
4. `PENDING_APPROVAL` is re-checked later, not blindly re-submitted.
5. `IN_PROGRESS` must be verified before retry.
6. `BLOCKED:MEMBERSHIP_REQUIRED` becomes retryable when membership reaches 1.
7. Membership blockers never suppress independently available posting.
8. Posting blockers never suppress membership.
9. Join questions are submitted once per actual join attempt; never repeatedly hammer Submit.

## Compliance

Do not fabricate user facts or bypass CAPTCHA, access controls, login challenges, group rules, administrator approval, platform restrictions, rate limits, or anti-spam protections. Do not use deceptive identities, account rotation, proxy rotation, or fingerprint evasion.

## Generic-only invariant

Core Postman files MUST remain business-agnostic. Specific brands, companies, industries, campaigns, products, jobs, locations, and identity facts may only appear as runtime input supplied for a particular execution, never as built-in defaults.

## End-of-run condition

Continue until every row has been scanned and both lanes evaluated where applicable, all remaining actions are non-actionable this run, a global hard blocker makes the session unusable, or a user-defined batch limit is reached.

Do not stop merely because some rows were completed.

## Final report

Return once at end:

- updated workbook;
- rows inspected;
- membership attempts/success/pending/failure/deferred;
- join-question forms auto-filled/submitted;
- rows blocked by unknown join-question facts;
- posting attempts/visible/pending-review/blocked/failure/deferred;
- rows where posting succeeded without membership;
- unresolved rows for next run;
- global blocker if any.

Do not interrupt the run with user-facing questions.
