# Membership Question Auto-Fill Policy

Postman 在 `UNATTENDED_BATCH` 模式下，遇到群组入群问题时应自动处理，不得默认停下来询问用户。

## 原则

使用三层答案来源，按优先级匹配：

1. **用户本轮明确提供的事实**；
2. **项目级固定答案库**；
3. **安全通用答案模板**。

只有当问题要求无法从以上来源确定的个人事实时，才标记：

```text
NEEDS_HUMAN:QUESTION_REQUIRES_USER_FACT
```

并继续 Posting Lane 与下一行，不得中断整批。

## 可自动回答的问题

### 同意规则

Question examples:

```text
Do you agree to follow the group rules?
Have you read the rules?
Will you respect the group rules?
```

Answer:

```text
Yes.
```

### 是否避免垃圾信息

Question examples:

```text
Will you avoid spam?
Do you agree not to post irrelevant content?
```

Answer:

```text
Yes.
```

### 加群目的

适用于招聘、外教、expat、jobs 等相关群组，且用户的内容分发目的与之匹配时：

```text
I work with StayChina, connecting qualified international educators with schools and education institutions in China. I would like to join the community, follow the group rules, and share relevant opportunities only where permitted.
```

如果当前 Postman 任务并非 StayChina 业务，则应根据用户本轮提供的组织/目的自动生成等价且真实的简短说明，不得虚构身份。

### 是否会遵守招聘规则

```text
Yes. I will follow the group rules and only share relevant opportunities where permitted.
```

### 是否会阅读置顶/公告

```text
Yes.
```

## 可从用户提供事实自动填写

只要本轮或项目配置中已经明确给出，可以直接填写：

- 姓名；
- 公司/组织名称；
- 网站；
- 联系方式；
- 所在城市/国家；
- 职业；
- 招聘业务说明；
- 是否为 recruiter / agency；
- 为什么加入；
- 计划发布什么类型内容。

回答必须严格基于已知事实。

## 不得猜测的问题

如果没有明确事实，不得自动编造：

- `Where do you currently live?`
- `What school do you work for?`
- `Who invited you?`
- `Which member do you know?`
- `How long have you lived in this city?`
- `What is your nationality?`
- `Are you a teacher?`
- 需要输入手机号末几位、生日、证件、账号验证等私人信息的问题。

处理：

```text
已加入=0
入群状态=NEEDS_HUMAN:QUESTION_REQUIRES_USER_FACT
```

但仍必须继续检查 Posting Lane；如果非成员也能发帖，则继续发布。

## 多选/勾选类问题

当选项含义明确且与真实业务匹配时可自动选择，例如：

```text
I agree to the rules
Job seeker
Recruiter
Education professional
Business / networking
```

只能选择真实匹配项。

不得为了入群而选择虚假身份，例如明明是 recruiter 却选择 teacher。

## 入群提交规则

自动填完所有可回答问题后：

1. 再检查一次答案是否基于已知事实；
2. 勾选必要的规则确认框；
3. 点击 `Submit` / `Join Group` / 等价按钮一次；
4. 根据可见结果更新：
   - 立即加入成功 → `已加入=1 / SUCCESS`
   - 已提交等待审核 → `已加入=0 / PENDING_APPROVAL`
   - 提交失败 → `已加入=0 / FAILED:<reason>`
5. 立即保存 Excel；
6. 不管入群结果如何，继续独立检查 Posting Lane。

## 无人值守要求

单个入群问题无法回答时：

```text
记录状态
→ 保存 Excel
→ 尝试 Posting Lane
→ NEXT ROW
```

禁止在单次运行中暂停并询问用户答案。
