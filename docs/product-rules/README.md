# 产品规则

此处存放 Cindy 客户端必须长期保持的产品行为、用户体验及跨端一致性要求。

## 收录标准

- 规则需说明“用户应获得何种结果”以及“哪些产品行为不可被破坏”。
- 每条规则应明确适用场景、产品不变量、原因、验收方法及允许的例外情况。
- 具体代码实现、命令及测试细节请置于 `docs/dev-rules/`，避免与产品目标混淆。
- 临时需求、待办事项及已有正式跟踪的问题不纳入永久产品规则。

## 当前规则

- [`core-product-principles.md`](core-product-principles.md)：阐述 Cindy 的宗旨、连接本质、Core 边界，以及 Agent、Skill、插件在产品层面的分工。
- [`review-product-direction.md`](review-product-direction.md)：定义 Cindy Review 的北极星目标、通用成果复核标准、运行与证据原则、当前边界、后续路线及待确认事项。
- [`region-and-editions.md`](region-and-editions.md)：说明 Global 与中国大陆版的关系、身份命名及默认值方向、区域相关 UI 标注与对外沟通口径。
- [`task-and-conversation-naming.md`](task-and-conversation-naming.md)：明确 `session` 的中文译名为「任务」，界定任务 / 对话 / 消息的分层定义与判定规则，并处理「任务」与 `task` 的歧义问题。