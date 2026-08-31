# Postman V2 Excel Schema

Postman V2 uses two independent workbooks:

1. **Group State Workbook** — execution state, membership, posting, history.
2. **Content Library Workbook** — indexed copy and campaign rules.

They are joined at runtime; do not merge them into one giant sheet.

---

## A. Group State Workbook

### Groups sheet

The original seven columns remain fixed at the left for backward compatibility:

| Col | Header | Type | Meaning |
|---|---|---|---|
| A | 序号 | Integer | Stable row/group ID |
| B | group名称 | Text | Group/community name |
| C | group url | URL | Target URL |
| D | 已加入 | 0/1 | Membership fact |
| E | 入群状态 | Text | Membership Lane state |
| F | 已发送 | 0/1 | Latest/current posting-state fact |
| G | 发送状态 | Text | Posting Lane state |

V2 appends the following columns after G:

| Header | Meaning |
|---|---|
| Group_Type | EXPAT / BUSINESS / VISA_HELP / TEACHER / SCHOOL / CITY_EXPAT / MOVING_TO_CHINA / GENERAL / UNKNOWN |
| Language | EN / ZH / ANY |
| Campaign_ID | Row-level campaign override; blank uses Run_Config |
| Last_Post_ID | Most recent attempted/successful post ID |
| Last_Post_Time | Timestamp of latest posting attempt/success as maintained by the agent |
| Next_Eligible_At | Earliest next posting time when repeat mode explicitly permits it |
| Send_Count | Cumulative actual posting attempts/successes according to implementation policy |
| Last_Post_URL | Published post URL when available |
| Last_Result | Most recent posting result |
| Failure_Reason | Most recent failure reason |
| Last_Checked_At | Last page inspection timestamp |
| Group_Rules_Summary | Short summary of visible posting rules relevant to this campaign |
| Promo_Allowed | YES / NO / UNKNOWN |
| Notes | Operator/agent notes |

### Legacy migration

Any workbook containing the original seven columns is valid input.

Postman must migrate it append-only:

- never delete/reorder A:G;
- preserve all existing statuses;
- append V2 columns if missing;
- create `Post_History` and `Run_Config` if missing;
- never reset an existing success merely to enable V2.

---

## B. Membership Lane states

| 入群状态 | 已加入 | Meaning |
|---|---:|---|
| PENDING | 0 | Not yet handled |
| IN_PROGRESS | 0 | Previous run may have been interrupted |
| SUCCESS | 1 | Joined successfully |
| SUCCESS:ALREADY_MEMBER | 1 | Page confirmed existing membership |
| PENDING_APPROVAL | 0 | Join request submitted |
| NOT_REQUIRED:POSTING_ALLOWED_WITHOUT_MEMBERSHIP | 0 | Posting available without membership |
| FAILED:<reason> | 0 | Membership attempt failed |
| SKIPPED:<reason> | 0 | Intentionally skipped |
| NEEDS_HUMAN:<reason> | 0 | Required fact/action unavailable this run |

Membership and posting remain independent. `已加入=0` does not imply posting is unavailable.

---

## C. Posting Lane states

| 发送状态 | 已发送 | Meaning |
|---|---:|---|
| PENDING | 0 | Not yet handled |
| IN_PROGRESS | 0 | Submission may be in flight/unknown |
| SUCCESS_VISIBLE | 1 | Post visibly published |
| SUCCESS_PENDING_REVIEW | 1 | Platform accepted post for moderation |
| BLOCKED:MEMBERSHIP_REQUIRED | 0 | Membership explicitly required |
| SKIPPED:RULE_PROHIBITS | 0 | Group rules prohibit current content |
| SKIPPED:NO_SUITABLE_COPY | 0 | No eligible Post_ID for this group/campaign |
| SKIPPED:ALREADY_SENT_THIS_CAMPAIGN | 0 | Campaign duplicate guard |
| SKIPPED:COOLDOWN | 0 | Not yet eligible under configured repeat mode |
| FAILED:<reason> | 0 | Submission failed |
| NEEDS_HUMAN:<reason> | 0 | Manual requirement exists |

`SUCCESS_PENDING_REVIEW` counts as sent.

---

## D. Post_History sheet

Append one row for every actual posting attempt.

Recommended columns:

1. Event_ID
2. Run_ID
3. 序号
4. Group_Name
5. Group_URL
6. Post_ID
7. Campaign_ID
8. Attempted_At
9. Result
10. Post_URL
11. Failure_Reason
12. Membership_State
13. Posting_State
14. Group_Type
15. Language
16. Notes

`Post_History` is the authoritative multi-run posting history. Do not store history inside the Content Library.

---

## E. Run_Config sheet

Recommended keys:

| Key | Example | Meaning |
|---|---|---|
| Active_Campaign_ID | COMPANY-SETUP | Default campaign |
| Default_Language | EN | Language fallback |
| Repeat_Mode | ONCE_PER_CAMPAIGN | Safe default |
| Default_Cooldown_Days | 30 | Used only in ROTATE_AFTER_COOLDOWN |
| Max_Posts_Per_Run | 0 | 0 means no user-defined cap |
| Dry_Run | FALSE | Inspect/match only if TRUE |
| Allow_Copy_Adaptation | MINIMAL | EXACT or MINIMAL |
| Require_Promo_Rule_Check | TRUE | Skip promotional content when rules prohibit it |
| History_Sheet | Post_History | History location |

Safe default:

```text
Repeat_Mode = ONCE_PER_CAMPAIGN
```

Optional:

```text
Repeat_Mode = ROTATE_AFTER_COOLDOWN
```

Only when group rules allow recurring promotional posts. Rotation must never be used to bypass anti-spam/platform enforcement.

---

# Content Library Workbook

## Copy_Index sheet

Required columns:

1. Post_ID
2. Campaign_ID
3. Status
4. Language
5. Audience_Type
6. Group_Type_Match
7. Geo_Match
8. Angle
9. Hook
10. Body_Copy
11. Landing_URL
12. Priority
13. Weight
14. Cooldown_Days
15. UTM_Content
16. Compliance_Note
17. Notes

Only `Status=ACTIVE` rows are eligible.

`Post_ID` is immutable and unique.

Normal Postman runs treat the Content Library as read-only.

## Campaigns sheet

Recommended columns:

- Campaign_ID
- Campaign_Name
- Status
- Default_Language
- Repeat_Mode
- Default_Cooldown_Days
- Landing_URL
- Objective
- Notes

## Match_Rules sheet

Recommended columns:

- Rule_ID
- Campaign_ID
- Group_Type
- Language
- Keyword_Hints
- Preferred_Post_IDs
- Priority
- Rule

This sheet tells Postman which copy is most suitable for which audience. It is not a hard one-group-one-post binding.

---

# Content-selection order

1. Resolve Campaign_ID.
2. Filter ACTIVE copy by campaign.
3. Match language.
4. Match group type/audience.
5. Apply geography when relevant.
6. Check group rules / Promo_Allowed.
7. Apply Match_Rules preferred IDs.
8. Apply duplicate/cooldown guard using Groups + Post_History.
9. Rank by Priority, Weight, least-recent use, then Post_ID.
10. Publish Body_Copy exactly or minimally adapt only when permitted.

---

# Fundamental invariants

```text
已加入=0  ≠  不能发帖
已加入=1  ≠  一定能发帖
Content Library = read-only during normal runs
Post_History = execution history
旧7列 = 永远保留在左侧
同一 Campaign 默认每群只成功发送一次
重复投放必须显式开启 cooldown 模式且符合群规
```
