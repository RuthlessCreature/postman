# Membership Question Auto-Fill Policy

Postman 是一个**完全通用**的群组/社区内容分发状态机，不预设任何公司、品牌、行业、地区、岗位或业务身份。

在 `UNATTENDED_BATCH` 模式下，遇到入群问题时应自动处理，不得默认停下来询问用户。

## 核心原则

所有答案只能来自当前任务明确提供的信息，不得从 Skill 本身假设具体业务。

答案来源按优先级为：

1. **当前任务明确提供的事实**；
2. **用户随 Excel / 帖子一起提供的可选身份与业务配置**；
3. **与具体身份无关的安全通用答案模板**。

任何业务专属答案都必须在当前任务输入中出现。Postman 本身不得内置或默认：

- 公司名；
- 品牌名；
- 网站；
- 电话/微信/邮箱；
- 招聘方身份；
- 教师身份；
- 所在城市；
- 国家/国籍；
- 行业；
- 职业；
- 组织关系；
- 加群目的。

只有当问题要求无法从当前输入确定的事实时，才标记：

```text
NEEDS_HUMAN:QUESTION_REQUIRES_USER_FACT
```

然后继续 Posting Lane 与下一行，不得中断整批。

## 可选任务级事实配置

用户可以在每次运行时附带一段任意格式的事实配置，例如：

```text
DISPLAY_NAME: <optional>
ORGANIZATION: <optional>
ROLE: <optional>
LOCATION: <optional>
COUNTRY: <optional>
WEBSITE: <optional>
CONTACT: <optional>
PURPOSE: <optional>
CONTENT_TYPE: <optional>
IS_RECRUITER: <optional true/false>
IS_BUSINESS: <optional true/false>
OTHER_FACTS: <optional>
```

这些字段全部是可选的，不要求固定格式，也不写入 Excel 七列。

Agent 只可使用实际提供的字段。未提供的字段视为未知，禁止猜测。

## 可安全自动回答的问题

### 同意群规

Question examples:

```text
Do you agree to follow the group rules?
Have you read the rules?
Will you respect the group rules?
```

当 Agent 已经实际读取/检查可见规则后，可以回答：

```text
Yes.
```

### 避免垃圾信息或无关内容

Question examples:

```text
Will you avoid spam?
Do you agree not to post irrelevant content?
Will you follow posting guidelines?
```

如果当前任务本身要求遵守群规，可以回答：

```text
Yes.
```

### 是否阅读公告/置顶规则

如果页面存在并且 Agent 已经实际检查，可以回答：

```text
Yes.
```

## 加群目的问题

Question examples:

```text
Why do you want to join this group?
What brings you to this community?
What is your purpose for joining?
```

### 有明确 PURPOSE / CONTENT_TYPE / 业务上下文

根据当前输入生成一个**最短、真实、中性**的回答。

通用结构：

```text
I would like to join this community to participate in relevant discussions and share content related to <known purpose/content type> where permitted. I will follow the group rules.
```

如果当前任务明确提供组织或角色，可自然加入，但不得为了显得可信而虚构。

### 没有明确目的事实

不得从帖子主题自行脑补用户身份。

如果只能确定“希望加入并遵守规则”，可以在问题允许泛化回答时使用：

```text
I would like to join the community and participate in relevant discussions while following the group rules.
```

如果问题明确要求具体身份/业务目的，而输入不足，则：

```text
NEEDS_HUMAN:QUESTION_REQUIRES_USER_FACT
```

## 可从当前输入自动填写的事实

只要用户在当前运行的帖子、说明、配置或其他明确输入中已经提供，就可以直接填写：

- 姓名/显示名；
- 公司/组织名称；
- 网站；
- 联系方式；
- 所在城市/国家；
- 职业/角色；
- 业务说明；
- 是否为 recruiter / agency / business；
- 为什么加入；
- 计划发布什么类型内容；
- 与群组主题的真实关系。

回答必须严格基于已知事实。

## 不得猜测的问题

如果没有明确事实，不得自动编造：

```text
Where do you currently live?
What company/school do you work for?
Who invited you?
Which member do you know?
How long have you lived here?
What is your nationality?
What is your profession?
Are you a teacher/recruiter/student/etc.?
What is your phone number?
```

以及任何要求：

- 生日；
- 证件；
- 手机号末位验证；
- 身份证明；
- 私密账号信息；
- 邀请人；
- 真实社会关系；
- 未知个人经历。

处理：

```text
已加入=0
入群状态=NEEDS_HUMAN:QUESTION_REQUIRES_USER_FACT
```

但仍必须继续检查 Posting Lane；如果非成员也能发布，则继续执行发布。

## 多选 / 单选 / 勾选问题

可自动选择的前提是：**选项与当前输入中的已知事实明确匹配。**

例如当前任务明确说明用户是 recruiter，才可以选择：

```text
Recruiter
```

当前任务明确说明是 job seeker，才可以选择：

```text
Job seeker
```

规则确认项可在已经实际检查规则后选择：

```text
I agree to the rules
```

不得为了通过审核而选择不真实的身份或关系。

## 自由文本问题的生成规则

自动生成回答时：

1. 优先 1–2 句；
2. 只使用当前输入中的事实；
3. 不增加营销宣传；
4. 不添加帖子里不存在的承诺；
5. 不虚构个人经历；
6. 不声称与管理员、成员、组织存在未知关系；
7. 语言尽量匹配问题语言；
8. 如果问题本身只要求确认，使用最短确认答案。

## 入群提交规则

自动填写所有可回答问题后：

1. 检查所有必填问题是否已获得真实答案；
2. 勾选可以真实确认的规则项；
3. 如果所有必填项均可回答，点击 `Submit` / `Join Group` / 等价按钮一次；
4. 根据可见结果更新：
   - 立即加入成功 → `已加入=1 / SUCCESS`
   - 已提交等待审核 → `已加入=0 / PENDING_APPROVAL`
   - 提交失败 → `已加入=0 / FAILED:<reason>`
5. 立即保存 Excel；
6. 不管 Membership Lane 结果如何，继续独立检查 Posting Lane。

如果某个**必填**问题无法真实回答：

```text
已加入=0
入群状态=NEEDS_HUMAN:QUESTION_REQUIRES_USER_FACT
```

不要提交带虚构答案的申请，并继续 Posting Lane / 下一行。

## 无人值守要求

单个入群问题无法回答时：

```text
记录状态
→ 保存 Excel
→ 尝试 Posting Lane
→ NEXT ROW
```

禁止在单次运行中暂停并询问用户答案。

## Generic-only invariant

Postman 仓库中的核心 Skill、Prompt、Schema、答案策略不得出现任何特定客户、品牌或业务作为默认逻辑。

特定业务只能作为**运行时输入**存在，不能成为 Postman 的内置假设。
