# Postman V2

通用的群组/社区内容分发状态机，面向具备浏览器操作能力的 Agent。

V2 的最终架构不是“一张大表塞所有东西”，而是两份 Excel：

```text
Group State Workbook                 Content Library Workbook
去哪发 / 当前状态 / 历史              发什么 / Campaign / Post_ID
        │                                      │
        └──────────── Postman runtime ─────────┘
                         │
                         ↓
             Campaign + 群类型 + 语言 + 历史
                         ↓
                    选择 Post_ID
                         ↓
              执行加入 / 发帖双轨
                         ↓
          写回 Groups + Post_History
```

## 为什么要拆成两份

群组表是执行数据库；文案库是母版。

- 换 Campaign，不需要改 1000 多行群组。
- 加一篇文案，只往 Copy Library 加一行。
- 群组执行失败，不污染文案库。
- 同一个群可以在不同 Campaign 中使用不同 Post_ID。
- `Post_History` 可以跨轮次防止重复发送。

## Group State Workbook

V2 保留原始七列在最左边，所以旧文件仍然能直接喂：

1. 序号
2. group名称
3. group url
4. 已加入
5. 入群状态
6. 已发送
7. 发送状态

Postman 会 append-only 自动追加：

- Group_Type
- Language
- Campaign_ID
- Last_Post_ID
- Last_Post_Time
- Next_Eligible_At
- Send_Count
- Last_Post_URL
- Last_Result
- Failure_Reason
- Last_Checked_At
- Group_Rules_Summary
- Promo_Allowed
- Notes

并创建：

- `Post_History`
- `Run_Config`

旧七列永不删除、永不换序。

模板：`postman_group_queue_v2_template.xlsx`

## Content Library Workbook

核心 Sheet：`Copy_Index`

每篇文案有唯一：

```text
Post_ID
Campaign_ID
```

并带：

- 受众类型
- 群类型匹配
- 语言
- 地域
- Angle
- Hook
- 完整 Body_Copy
- Landing URL
- Priority / Weight
- Cooldown
- Compliance Note

另外包含：

- `Campaigns`
- `Match_Rules`
- `Library_Config`

正常运行时 Content Library **只读**。

模板：`post_copy_library_v2_template.xlsx`

## V2 选择逻辑

Postman 不要求你给每个群手工绑定一篇帖子。

它会：

```text
resolve Campaign
  ↓
match Group_Type + Language + Geo
  ↓
apply Match_Rules
  ↓
check Post_History / cooldown
  ↓
rank Priority + Weight
  ↓
select Post_ID
  ↓
publish Body_Copy
```

### 默认重复策略

安全默认：

```text
ONCE_PER_CAMPAIGN
```

同一个群、同一个 Campaign 成功发过后，不自动再发。

如果确实需要长期持续投放，可以显式启用：

```text
ROTATE_AFTER_COOLDOWN
```

但只有群规允许商业/推广内容并且 cooldown 已到期才允许再次发送。

Postman 不使用换文案、换账号、代理、时间伪装等方式绕过平台反垃圾或群规。

## 最重要的设计仍然不变：双轨独立

每个 group 同时评估：

- Membership Lane：能不能/需不需要加入；
- Posting Lane：现在能不能直接发。

```text
已加入=0 ≠ 不能发帖
```

有些群允许非成员投稿，所以入群审核中也可能已经成功发帖。

只有页面明确要求 membership 时，Posting Lane 才依赖 Membership Lane。

## 无人值守

单次运行中：

- 不逐群问用户；
- 入群问题能根据已知真实事实回答就自动填写；
- 无法真实回答的写 `NEEDS_HUMAN` 后继续；
- 群规禁止推广就跳过；
- 一个群失败不影响下一行；
- 每个真实 side effect 后 checkpoint；
- 所有实际发帖尝试追加到 `Post_History`；
- 最后返回更新后的 Group State Workbook。

## 推荐调用

```text
Use $postman in V2 unattended mode.
Load the group-state workbook and the content-library workbook.
Auto-migrate legacy seven-column group queues append-only.
Treat membership and posting as independent lanes.
Resolve Campaign_ID, match group type/language, select an eligible Post_ID using Match_Rules and Post_History, respect group rules and duplicate/cooldown guards, publish only when permitted, append Post_History, checkpoint after side effects, and return the updated group-state workbook.
```

详细字段见 `SCHEMA.md`。

## 非目标

Postman 不负责绕过 CAPTCHA、登录验证、群规、管理员审核、平台限制、频率限制或反垃圾机制，也不伪造入群问题答案。
