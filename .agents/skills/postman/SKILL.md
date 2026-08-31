---
name: postman
description: Generic unattended, resumable group/community posting workflow driven by a group-state Excel workbook plus an optional indexed content-library Excel workbook. Membership and posting are independent state machines. Legacy seven-column queues are auto-migrated append-only. Content is selected at runtime by Campaign_ID, group type, language, match rules and posting history. Process all actionable rows without mid-run user interaction, checkpoint after side effects, append posting history, defer row-level blockers, and resume from the updated workbook next iteration.
---

# Postman V2

## Purpose

Run an **unattended, resumable, business-agnostic content-distribution workflow** across groups or communities.

Postman V2 separates two kinds of state:

1. **Group State Workbook** — where to go, membership state, posting state, last post, cooldown and history.
2. **Content Library Workbook** — what may be posted, indexed by `Post_ID` and `Campaign_ID`.

The two workbooks are not manually bound row-by-row. Postman joins them at runtime by campaign, group type, language, matching rules and history.

Postman MUST NOT assume any specific company, brand, industry, campaign, geography, identity or posting purpose. Those facts come only from runtime input and the supplied workbooks.

The user does not intervene during a run. Process every currently actionable row, persist all results, defer unresolved work and return the updated **Group State Workbook**. The next run resumes from that workbook.

Before execution, read `MEMBERSHIP_ANSWERS.md` when available.

---

# 1. Accepted input modes

## Preferred V2 mode

Required:

1. One Group State `.xlsx` workbook.
2. One Content Library `.xlsx` workbook.

Optional:

- runtime facts for join questions;
- an explicit `Campaign_ID` override;
- a user-defined batch limit;
- dry-run instruction.

## Legacy compatibility mode

Postman MUST also accept the original seven-column workbook plus one user-supplied content payload.

Legacy columns, in this exact order:

1. `序号`
2. `group名称`
3. `group url`
4. `已加入`
5. `入群状态`
6. `已发送`
7. `发送状态`

A legacy workbook is automatically upgraded **append-only**. Never delete, rename or reorder these original seven columns.

If a Content Library is not supplied, use the single runtime content payload and preserve legacy duplicate-prevention semantics.

---

# 2. Group State Workbook contract

## Required legacy columns A:G

These remain the compatibility core and MUST stay in this order:

| Col | Header | Meaning |
|---|---|---|
| A | `序号` | Row/group identifier |
| B | `group名称` | Group/community name |
| C | `group url` | Target URL |
| D | `已加入` | Membership fact: `0/1` |
| E | `入群状态` | Membership Lane state |
| F | `已发送` | Latest/current posting-state fact: `0/1` |
| G | `发送状态` | Posting Lane state |

## V2 columns appended after G

When missing, append these columns without changing A:G:

- `Group_Type`
- `Language`
- `Campaign_ID`
- `Last_Post_ID`
- `Last_Post_Time`
- `Next_Eligible_At`
- `Send_Count`
- `Last_Post_URL`
- `Last_Result`
- `Failure_Reason`
- `Last_Checked_At`
- `Group_Rules_Summary`
- `Promo_Allowed`
- `Notes`

When missing, also create:

- `Post_History`
- `Run_Config`

Migration rules:

1. append only;
2. preserve every existing group row and status;
3. preserve `已发送=1`; never reset it merely because V2 is enabled;
4. create new columns/sheets with blank/default values;
5. checkpoint the migrated workbook before external side effects when practical.

---

# 3. Content Library Workbook contract

Normal runs treat the Content Library as **read-only**.

Required `Copy_Index` columns:

- `Post_ID`
- `Campaign_ID`
- `Status`
- `Language`
- `Audience_Type`
- `Group_Type_Match`
- `Geo_Match`
- `Angle`
- `Hook`
- `Body_Copy`
- `Landing_URL`
- `Priority`
- `Weight`
- `Cooldown_Days`
- `UTM_Content`
- `Compliance_Note`
- `Notes`

Recommended sheets:

- `Copy_Index`
- `Campaigns`
- `Match_Rules`
- `Library_Config`

Only rows with `Status=ACTIVE` are eligible.

`Post_ID` is immutable. Never reuse one `Post_ID` for materially different copy.

Do not mutate `Body_Copy`, `Status`, priorities or library history during normal distribution runs.

---

# 4. Campaign resolution

Resolve the active campaign using this precedence:

1. explicit user/runtime `Campaign_ID` override;
2. nonblank `Groups.Campaign_ID` for that row;
3. `Run_Config.Active_Campaign_ID`;
4. Content Library default campaign if defined.

If no campaign can be resolved in V2 library mode:

```text
发送状态=NEEDS_HUMAN:CAMPAIGN_NOT_RESOLVED
```

Continue the Membership Lane and then the next row.

---

# 5. Content selection engine

For an eligible group row, select copy deterministically.

## Step A — filter

Filter `Copy_Index` by:

1. `Campaign_ID` matches active campaign;
2. `Status=ACTIVE`;
3. `Language` matches group language, or copy language is `ANY`;
4. `Group_Type_Match` contains the resolved group type, or a generic/ANY match is explicitly allowed;
5. `Geo_Match` matches known geography when location-specific copy is required, otherwise `ANY`;
6. group rules permit that type of promotional/commercial content.

Do not force a weak audience match merely to publish something.

## Step B — apply Match_Rules

When `Match_Rules` contains a rule for the campaign/group type/language, use its `Preferred_Post_IDs` as the preferred candidate set.

Keyword hints may classify a group from its name or visible description when `Group_Type` is blank or `UNKNOWN`.

Do not invent group geography, profession, membership or audience facts that are not visible or supplied.

## Step C — duplicate and cooldown guard

Consult both:

- `Groups.Last_Post_ID / Last_Post_Time / Next_Eligible_At`;
- `Post_History` for the same group URL or row identifier.

Default safe mode is:

```text
ONCE_PER_CAMPAIGN
```

In this mode, if the same group already has a successful `Post_History` event for the active `Campaign_ID`, do not post again unless the user starts a new campaign or explicitly resets the campaign state.

Optional mode:

```text
ROTATE_AFTER_COOLDOWN
```

Only use this when:

- runtime/Run_Config explicitly enables it;
- group rules allow recurring promotional posts;
- `Next_Eligible_At <= now`;
- the selected `Post_ID` is not still inside its cooldown;
- repeated posting does not conflict with visible group rules.

Postman MUST NOT use rotation, alternate copy, account changes or timing tricks to evade anti-spam or platform enforcement.

## Step D — rank

Among remaining candidates:

1. Match_Rules preferred order;
2. lower numeric `Priority` first;
3. higher `Weight` first;
4. least recently used for that group/type when history is available;
5. avoid `Last_Post_ID` when another equally suitable candidate exists;
6. final deterministic tie-break: lexical `Post_ID`.

## Step E — materialize content

Use `Body_Copy` as the source text.

- `EXACT` mode: publish exactly as stored.
- `MINIMAL` mode: only adapt formatting or a small local phrase when visible group rules require it.

Never change material facts, promises, pricing, identity, location, legal claims, contact information or landing URL unless the supplied runtime facts explicitly authorize the change.

---

# 6. Core model: TWO INDEPENDENT LANES

Every group row has two independent state machines.

## Membership Lane

Controls only:

- `已加入`
- `入群状态`

## Posting Lane

Controls posting eligibility and attempt state.

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

---

# 7. Membership states

| Outcome | 已加入 | 入群状态 |
|---|---:|---|
| Already member | 1 | `SUCCESS:ALREADY_MEMBER` |
| Join succeeds | 1 | `SUCCESS` |
| Join request submitted | 0 | `PENDING_APPROVAL` |
| Posting allowed without membership | 0 | `NOT_REQUIRED:POSTING_ALLOWED_WITHOUT_MEMBERSHIP` |
| Join fails | 0 | `FAILED:<reason>` |
| Unknown fact/manual action required | 0 | `NEEDS_HUMAN:<reason>` |
| Join intentionally skipped | 0 | `SKIPPED:<reason>` |

---

# 8. Posting states

| Outcome | 已发送 | 发送状态 |
|---|---:|---|
| Post visible | 1 | `SUCCESS_VISIBLE` |
| Accepted for moderation | 1 | `SUCCESS_PENDING_REVIEW` |
| Membership required | 0 | `BLOCKED:MEMBERSHIP_REQUIRED` |
| Rule prohibits current content | 0 | `SKIPPED:RULE_PROHIBITS` |
| No suitable copy | 0 | `SKIPPED:NO_SUITABLE_COPY` |
| Not yet eligible due to configured cooldown | 0 | `SKIPPED:COOLDOWN` |
| Submission failed | 0 | `FAILED:<reason>` |
| Manual action required | 0 | `NEEDS_HUMAN:<reason>` |

`SUCCESS_PENDING_REVIEW` MUST set `已发送=1`.

In V2, `已发送` describes the current/latest posting state and is not the historical database. `Post_History` is the authoritative record of all posting attempts.

---

# 9. Post_History contract

Append one row for every actual posting attempt, successful or failed.

Recommended fields:

- `Event_ID`
- `Run_ID`
- `序号`
- `Group_Name`
- `Group_URL`
- `Post_ID`
- `Campaign_ID`
- `Attempted_At`
- `Result`
- `Post_URL`
- `Failure_Reason`
- `Membership_State`
- `Posting_State`
- `Group_Type`
- `Language`
- `Notes`

Do not append a posting event for a row that was only inspected and never submitted.

After a posting attempt, update the Groups row:

- `Last_Post_ID`
- `Last_Post_Time`
- `Send_Count`
- `Last_Post_URL` when known
- `Last_Result`
- `Failure_Reason`
- `Last_Checked_At`
- `Next_Eligible_At` when repeat/cooldown mode is in use

---

# 10. Start-of-run recovery

- `已加入=1` → never join again unless user resets it.
- `PENDING_APPROVAL` → re-check approval; never blindly resubmit.
- `IN_PROGRESS` → verify real state before retry.
- `BLOCKED:MEMBERSHIP_REQUIRED` → retry posting only if membership is now confirmed.
- transient `FAILED:*` → at most one active retry per run.
- `NEEDS_HUMAN:*` → passively re-check once; if unresolved, preserve and continue.
- legacy `已发送=1` with no history → treat as a duplicate guard; do not blindly repost.
- V2 successful history for active campaign → respect campaign repeat rules before attempting again.

---

# 11. Per-row algorithm

## Step 1 — validate and classify

Resolve:

- group URL/name;
- membership state;
- group type;
- language;
- active campaign;
- visible group rules;
- whether promotional/commercial posts are allowed;
- whether posting is currently available without membership.

If `Group_Type` is blank/UNKNOWN, use library `Match_Rules.Keyword_Hints` plus visible group evidence to classify conservatively. Save the classification only when reasonably grounded.

Set `Last_Checked_At`.

## Step 2 — advance Membership Lane

If `已加入=0`:

1. already member → `1 / SUCCESS:ALREADY_MEMBER`;
2. else if Join is available, start the normal join flow once;
3. answer join questions using the auto-fill rules below;
4. submit only when every required answer is truthfully grounded;
5. immediate success → `1 / SUCCESS`;
6. admin review → `0 / PENDING_APPROVAL`;
7. failure → `0 / FAILED:<reason>`;
8. if joining is unnecessary/unavailable while posting is permitted → `0 / NOT_REQUIRED:POSTING_ALLOWED_WITHOUT_MEMBERSHIP`;
9. checkpoint after each material change.

Do not continue to the next row merely because membership remains 0. Posting Lane still must be evaluated.

## Step 3 — resolve copy

If the Posting Lane is potentially actionable:

1. resolve campaign;
2. apply content-library filters;
3. inspect successful history/cooldown;
4. select `Post_ID` deterministically;
5. load `Body_Copy`;
6. if no suitable copy remains → `SKIPPED:NO_SUITABLE_COPY`.

## Step 4 — advance Posting Lane

If posting is eligible:

1. if group rules prohibit the content → `SKIPPED:RULE_PROHIBITS`;
2. if current campaign has already succeeded and mode is `ONCE_PER_CAMPAIGN` → `SKIPPED:ALREADY_SENT_THIS_CAMPAIGN`;
3. if cooldown applies → `SKIPPED:COOLDOWN`;
4. if composer is available, publish even when `已加入=0`;
5. if page explicitly requires membership → `BLOCKED:MEMBERSHIP_REQUIRED`;
6. before submission set `发送状态=IN_PROGRESS` and checkpoint if practical;
7. insert selected `Body_Copy`;
8. submit once;
9. verify outcome;
10. append `Post_History` event;
11. update Groups V2 tracking fields;
12. visible → `1 / SUCCESS_VISIBLE`;
13. accepted for review → `1 / SUCCESS_PENDING_REVIEW`;
14. failed → `0 / FAILED:<reason>`;
15. checkpoint and continue.

## Step 5 — explicit dependency reconciliation

If Posting Lane was `BLOCKED:MEMBERSHIP_REQUIRED` and Membership Lane reaches `已加入=1` in the same visit, re-check posting immediately and publish if otherwise eligible.

---

# 12. Join Question Auto-Fill

Answer membership questions automatically using this precedence:

1. facts explicitly supplied in the current task;
2. runtime identity/business/context facts supplied with this run;
3. `MEMBERSHIP_ANSWERS.md` approved generic templates.

The Skill itself has no built-in organization, role, business, industry, location or campaign identity.

## Generic rule questions

Examples:

- `Do you agree to the group rules?` → `Yes.` only after the rules were actually checked.
- `Will you avoid spam / irrelevant posts?` → `Yes.`
- `Will you follow posting guidelines?` → `Yes.`

## Purpose questions

Generate a short truthful response from supplied purpose/content facts.

If no specific purpose is supplied and a generic community-participation answer is acceptable:

```text
I would like to join the community and participate in relevant discussions while following the group rules.
```

## Unknown factual question

If a required question asks for an unknown identity, affiliation, location, occupation, inviter, personal history or verification fact:

```text
已加入=0
入群状态=NEEDS_HUMAN:QUESTION_REQUIRES_USER_FACT
```

Do not guess. Do not pause to ask the user. Continue evaluating Posting Lane.

---

# 13. Unattended batch mode

During one run:

- do not ask the user questions;
- do not request per-group approval;
- do not request posting confirmation;
- do not stop for join questions;
- do not stop because one row or lane is blocked;
- process all automatically actionable rows;
- convert unresolved work into spreadsheet state;
- checkpoint after external side effects;
- return the updated Group State Workbook once at end.

Human intervention, when needed, happens between runs.

---

# 14. Row-level vs global blockers

Row-level blockers MUST NOT terminate the batch:

- group unavailable;
- join pending/denied;
- unknown required join answer;
- posting disabled;
- group rules prohibit promotional content;
- no suitable copy in the active campaign;
- campaign/cooldown makes this row ineligible;
- one lane fails while the other can proceed.

Global blockers may stop external actions only when the whole session becomes unusable:

- session-wide login/device verification;
- CAPTCHA blocking all further actions;
- account-wide posting restriction;
- browser/computer-use capability unavailable.

Before stopping: checkpoint current workbook and return it.

---

# 15. Checkpoint discipline

The Group State Workbook is the persistent execution database.

Checkpoint after every material side effect, including:

- join-question submission;
- join request/result;
- post submission/result;
- history append;
- material state transition.

If the environment only permits a final file write, maintain an exact mutation log and apply every mutation before returning.

The Content Library remains read-only.

---

# 16. Compliance

Do not fabricate user facts or bypass CAPTCHA, access controls, login challenges, group rules, administrator approval, platform restrictions, rate limits or anti-spam protections.

Do not use deceptive identities, account rotation, proxy rotation, fingerprint evasion, hidden timing tricks or copy rotation for the purpose of defeating platform enforcement.

Promotional content must be skipped where visible group rules prohibit it.

Copy rotation is for audience relevance and duplicate control, not evasion.

---

# 17. Final report

Return once at end:

- updated Group State workbook;
- rows inspected;
- rows migrated to V2 if any;
- membership attempts/success/pending/failure/deferred;
- join-question forms auto-filled/submitted;
- posting attempts/visible/pending-review/blocked/failure/deferred;
- posts skipped by rules, campaign duplicate guard or cooldown;
- `Post_ID` usage counts;
- rows where posting succeeded without membership;
- unresolved rows for next run;
- global blocker if any.

Do not interrupt the run with user-facing questions.
