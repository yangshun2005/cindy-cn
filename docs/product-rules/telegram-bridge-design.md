# Telegram 统一消息桥接设计

> 状态：设计稿，供客户端实现与跨仓协议评审使用。本文不是已上线能力清单。
>
> 目标：让官方 bot 与个人 bot 在用户能感知的消息过程、终稿收口和能力降级上遵守同一套
> 消息语义，同时保留两条入口在身份、绑定、权限、群触发和可靠性上的必要差异。
>
> 本文只设计客户端边界，不修改服务端、`cindy-protocol`、数据库 schema 或线上行为。
> 文中标为“目标”的内容不应被读成当前已经接线。

## 1. 先给结论

最终形态不是两份互相复制的 Telegram 消息系统，也不是把两个 bot 强行塞进一个带条件分支
的 adapter。应采用一份共享的消息内核，加上两个很薄的传输出口：

```text
                    ┌──────────────────────────┐
                    │  Telegram 消息内核        │
                    │  事件 → 呈现 → 生命周期   │
                    │  终稿栅栏 → 回执/重试      │
                    └────────────┬─────────────┘
                                 │ provider-neutral intents
                    ┌────────────┴─────────────┐
                    │                          │
       个人 bot adapter                    官方 bot adapter
       Telegram Bot API                    msg.op / legacy frames
       owner + local token                 binding + principal + server auth
```

共享内核拥有“这一轮消息应该经历什么状态”；adapter 只拥有“怎样把一个意图投递到对应
的 Telegram 连接”。官方 adapter 在新服务端支持时使用细粒度 `msg.op`，在旧服务端或能力
未协商时继续使用 `turn.progress` / `turn.end` 回退。个人 adapter 直接调用 Bot API。

这会解决“最后一个答案一直等、迟到的进度又把终稿改回去、两条代码越维护越不一样”这类
客户端问题，但不会假装客户端可以替服务端完成官方 bot 的授权和消息操作。官方真正的
`send` / `edit` / `delete` / `typing` / `media` 仍须等服务端提供并声明对应能力。

### 1.1 当前实现与目标设计的边界

| 范围              | 当前已经存在                                                                                      | 本设计要达到的目标                                                   |
| ----------------- | ------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| 文本与过程合成    | `apps/desktop/src/main/im/shared/turnPresenter.ts`、`turnActivity.ts` 已共享                      | 由内核继续作为唯一过程视图来源；不把两种正文累积模式误合成一种       |
| 个人 bot 过程消息 | `sendMessage` 建首条真实消息，之后 `editMessageText` 节流更新；终稿有原位编辑失败后的 repost 兜底 | 统一为“过程载体”和“新鲜终稿”两个明确阶段；删除旧载体不阻塞答案       |
| 官方 bot 过程消息 | 主要由 `turn.progress` / `turn.end` 驱动；桌面端目前只真正消费 `msg.op` 的 `react`                | 用官方 adapter 兑现同一内核意图；未协商时保留旧帧回退                |
| 终稿              | 官方桌面已有本地 `requestLedger` 持久终稿账本；服务端不保存正文/outbox；个人主要是进程内尽力投递  | 终稿发送独立于过程更新，建立终稿栅栏、幂等键和不确定回执处理         |
| 草稿              | 个人曾试用原生草稿，因只能承载一行纯文本而撤回；当前个人 DM 也走消息流                            | 只有声明支持完整过程内容的 DM draft 能力时才启用；草稿永远不承载终稿 |
| 消息效果          | 官方服务端已有 DM 终稿 `messageEffectId`；个人能力契约为不支持                                    | 以可选装饰意图下发；失败只降级装饰，不影响正文送达                   |
| 数据库            | 群消息库与保留核心已共享，但 provider namespace 分开                                              | 保留 namespace 隔离和消息所有权；本设计不改 schema                   |

### 1.2 与已合并的 #2305 的关系

PR #2305（官方 Telegram 进度呈现与完成收口）已经合并。它补的是官方 `turn.end` 之前
的最后一次进度 flush，防止 observer 的尾沿节流在 teardown 时吞掉最后快照；它不是
`msg.op` 消息桥，也没有让官方 bot 具备个人 bot 的完整终稿生命周期。

本设计把 #2305 的 flush 保留为**旧帧 adapter 的兼容回退**，不回退、不重复实现。未来
官方 adapter 切到 `msg.op` 后，终稿栅栏仍要先阻止迟到的 `turn.progress`，而不是依赖
“再 flush 一次”来保证答案。

## 2. 术语与不可变身份

### 2.1 用户可见术语

面向用户仍使用“任务、对话、消息”的既有边界；`turn` 是内部“一轮”，不要在 UI 文案里
把一轮称为新的产品对象。过程载体、终稿和交互卡都是消息呈现阶段，不是额外的任务。

### 2.2 一轮的身份键

内核不直接接受裸 `chatId` 或裸消息 ID。每个事件和出站意图都带有以下逻辑身份（具体
wire 字段由未来协议另行裁定）：

```ts
type TelegramRoundIdentity = {
  provider: "telegram-personal" | "telegram-official";
  accountId: string; // personal botId；official connection/account
  scope: {
    kind: "dm" | "group";
    chatId: string;
    deliveryThreadId: string | null; // Telegram 投递位置，可是 reply root
    topicId: string | null; // 仅 is_topic_message 才是归属 topic
    replyToMessageId: string | null; // 本轮受理时冻结的回挂锚点
  };
  principalId: string | null; // official 必须来自触发者；personal 可为空
  bindingId: string | null; // official 绑定代际
  bindingGeneration: number;
  connectionGeneration: number;
  roundId: string;
};
```

`deliveryThreadId` 与 `topicId` 不能互换。Telegram 普通群的 reply 链也可能带
`message_thread_id`；它决定消息投到哪里，但只有 `is_topic_message === true` 才能决定
它属于哪个 topic lane。

`replyToMessageId` 在本轮被受理时冻结。过程消息、交互卡和终稿都从这份 round identity
取回挂目标，不能在发送终稿时重新从“最近群消息”或可变队列领取；这是防止群友新发言把
最后答案挂错位置、看起来像被替换的关键不变量。

内核生成的每个意图还带 `sequence`（同一 `roundId` 单调递增）和 `phase`。adapter 必须
拒绝不匹配的 `bindingGeneration` / `connectionGeneration`，以及终稿栅栏之后的过程更新。

## 3. 身份、主人、群友和陌生人路由矩阵

下表描述**现有行为必须保留**的身份差异；统一的是消息内核，不是权限模型。

| 入口与来信                                     | lane / 身份                                                                        |                        是否创建任务 | 允许的执行范围                                                             | 用户可见响应                                      |
| ---------------------------------------------- | ---------------------------------------------------------------------------------- | ----------------------------------: | -------------------------------------------------------------------------- | ------------------------------------------------- |
| 个人 bot 私聊，bot owner                       | `telegram-personal:<botId>` + user chat                                            |                                  是 | 按个人 bot 的 owner 权限与本地设置                                         | 正常过程消息、终稿、交互卡                        |
| 个人 bot 私聊，非 owner 或未通过 auth          | 仍按该 bot 的 chat 隔离                                                            | 由个人 auth/policy 决定；不通过则否 | 不得越过 owner / channel policy                                            | 现有陌生人或权限提示；不把拒绝伪装成已执行        |
| 个人 bot 群内正确 `@`、回复 bot 或有效命令     | 一群（及 topic）共享 per-chat lane                                                 |                                  是 | 群轮次沿用个人 bot 的低权限/强制确认策略；不因统一内核改成 owner bypass  | 过程消息与终稿发回原群/原 reply 位置              |
| 个人 bot 群友/陌生人普通发言（`mention` 模式） | 写入共享群窗口，不取得 owner 身份                                                  |                                  否 | 仅作同群上下文；不能借 owner 轮次访问其它群或私聊                          | 无主动回复                                        |
| 个人 bot 群 `always` 的 ambient 消息           | 同一群 lane，`ambient=true`                                                        |                                是（若正文有效） | 同上                                                                       | 不 typing、不打过程表情；模型可用 `NO_REPLY` 静默 |
| 官方 bot 私聊，已绑定 principal                | `telegram:<principalId>`                                                           |                                  是 | 路由到该 principal 当前 `bindingId` 的 Cindy 桌面；使用官方既有权限口径    | 由绑定桌面驱动过程、交互和终稿                    |
| 官方 bot 私聊，未绑定 principal                | 无有效 binding                                                                     |                                  否 | 不创建任务、不猜测其他人的桌面                                             | 限频关联/绑定提示                                 |
| 官方 bot 群内，已绑定 principal 正确 `@`       | 主群 `telegram:group:…:<principal>`；topic `telegram:topic:…:<thread>:<principal>` |                                  是 | 只进入发言者自己的 binding；不把被 @ 的其他群友当主人                      | 回到该群；relay 的其他副本不等于该轮的回答        |
| 官方 bot 群内，未绑定发言者 `@`                | 无有效 binding                                                                     |                                  否 | 不创建任务，不替未绑定用户执行                                             | 限频关联提示；不得把提示当作任务结果              |
| 官方群友/陌生人普通发言（未触发）              | 可能成为 `group.message` relay 上下文                                              |                                  否 | 只能投给该群已登记且 recipient 代际匹配的桌面；不取得任何 binding 的执行权 | 无主动回复                                        |
| 官方群 relay 给同群其他已登记桌面              | `group.message` + 精确 `recipient{bindingId,principalId}`                          |  否（除非该桌面随后收到自己的触发） | 只作为本地群窗口/上下文；接收方代际必须精确匹配                            | 通常无 Telegram 出站；不会覆盖别人的终稿          |
| 任一 bot 的 topic/reply                        | `topicId` 由归属判据决定，裸 thread 只用于投递                                     |                      按上行触发规则 | topic lane 与主群 lane 隔离；普通 reply 不得凭空新建 topic lane            | 终稿沿用原 reply/topic 目标                       |
| 连接重连、换绑、解绑后迟到帧                   | 原 round 的旧 generation                                                           |                                  否 | fail-closed；不向新主人或新 binding 写消息                                 | 旧过程载体可留在原聊天，但新终稿不能被旧帧覆盖    |

### 3.1 “官方 bot 关联任何被 @ 的人”具体含义

官方 bot 的 `principalId` 是**发言人的 Telegram user ID**。`@Cindy` 不是把消息发送给
被 @ 的任意群友，而是把这条触发路由到发言人自己绑定的 Cindy 桌面。群中其他已登记桌面
可能收到 relay 上下文，但它们不是这条任务的 owner，也不能借 relay 结果回写回答。

个人 bot 则是用户自持 token 的低权限入口；群里多人共用一条 per-chat lane 的既有行为
不能被官方 principal lane 的规则覆盖。

## 4. 共享消息内核与两个 adapter 的边界

### 4.1 内核负责什么

共享内核是纯客户端业务层，至少包含以下职责：

1. 把 thinking、工具调用、正文增量、交互请求和终态转换为统一的活动模型；
2. 使用 `turnPresenter` / `turnActivity` 的同一过程视图规格，保留
   `finalized-segments` 与 `buffer-replace` 两种明确正文模式；
3. 管理过程载体（draft 或过程消息）、终稿消息、交互卡和 typing 的生命周期；
4. 只生成 provider-neutral 的 `send`、`edit`、`delete`、`react`、`typing`、`media`、
   `effect`、`flush`、`reconcile` 意图；
5. 执行长度限制、分段、Markdown/HTML/Rich Message 分层和降级顺序；
6. 在终稿栅栏后丢弃迟到的过程帧，处理 `NO_REPLY`、取消、失败、断线和不确定回执；
7. 给每个意图分配稳定的 `roundId + phase + sequence`，保证幂等和所有权可追踪；
8. 把可测试的结果状态交给 adapter，不把“API 调用返回了”误当成“用户已经看到”。

内核不负责 Telegram token、服务端 binding、principal 授权、群成员证据、服务端 relay
扇出或消息数据库 schema。

### 4.2 过程区与工具调用的用户可见规则

两个 bot 的过程内容必须来自同一套 presenter 输出，不允许 adapter 自己拼另一份“更像
Telegram”的时间线：

- thinking、工具调用和正文按实际发生顺序进入同一轮；同一个 thinking block 原位更新，
  同一个 `toolUseId` 在 replay/重连后去重；
- 工具调用显示共享的人类可读摘要，不直接倾倒整段 JSON、命令参数或可能含凭证的原始输入；
- 默认展示最近五个可读步骤和总步骤数，正文继续显示在过程区下方；等待授权、等待回答、
  自动重试与“正在写”是明确状态，不伪装成一个新的工具步骤；
- 稠密事件按同一 trailing-edge 规格合帧；完全相同的快照不重复编辑，前缀相似不能被误判
  为相同，否则正文增量会看起来卡死；
- DM draft 若不能完整承载这些步骤，就必须降级为真实过程消息，不能只显示一行“思考中”；
- 成功终稿只包含正文；失败是否保留过程现场、ambient 是否静默继续按 parity 台账裁决，
  不在 presenter 里偷做统一。

adapter 只决定这些内容放进 draft、真实过程消息还是 legacy progress 帧，不得改变顺序、
文案摘要、去重和终态语义。

### 4.3 个人 bot adapter 负责什么

- 持有 Bot API client、token、owner 和本地 `configVersion`；
- 解析 Telegram 入站的 `is_topic_message`、reply、media 和群 activation；
- 把内核意图投到 `sendMessage` / `editMessageText` / `deleteMessage` /
  `sendChatAction` / media API；
- 维护本地 reply target、typing loop 和个人 bot 的 outbox/重试边界；
- 以 `telegram-personal:<botId>` 隔离群消息窗口和消息所有权；
- 在每个真实出站 await 前验证本轮 owner 与 connection generation 仍有效。

个人 adapter 不能因为共享内核而获得官方 bot 的 principal 绑定语义，也不能把群成员的
消息自动提升为 owner 权限。

### 4.4 官方 bot adapter 负责什么

- 把入站 `task.dispatch` / `group.message` 解释成官方 principal/binding 上下文；
- 在 `recipient{bindingId,principalId}`、当前 binding 和 generation 不一致时丢弃；
- 新能力可用时经 `msg.op` 请求服务端执行 Telegram 出站；
- 未协商或旧服务端时回退到现有 `turn.progress` / `turn.end`；
- 处理官方桌面本地 `requestLedger` 终稿账本、ACK、客户端重投和 24 小时终稿时效；服务端
  不取得正文 outbox 所有权；
- 保留官方 `/status`、`/unlink`、群 onboarding 和 per-principal lane；
- 不把客户端自带的 `chatId` 当授权依据，所有发送目标必须来自服务端授权上下文。

官方 adapter 不能把个人 bot 的本地 token、群低权限模型或 per-chat lane 带入官方出口。

### 4.5 内核意图的最小形状（设计，不是当前协议）

```ts
type BridgeIntent = {
  round: TelegramRoundIdentity;
  phase: "process" | "final" | "cleanup" | "interaction" | "decoration";
  sequence: number;
  deliveryKey: string; // roundId + phase + logical part；重试不变
  op:
    | { kind: "send-process"; content: RenderedContent }
    | { kind: "update-process"; carrier: CarrierRef; content: RenderedContent }
    | { kind: "send-final"; content: RenderedContent; effect?: string }
    | { kind: "clear-process"; carrier: CarrierRef }
    | { kind: "delete-process"; carrier: CarrierRef }
    | { kind: "typing"; active: boolean }
    | { kind: "react"; target: InboundMessageRef; emoji: string }
    | { kind: "media"; content: MediaContent }
    | { kind: "reconcile"; deliveryKey: string };
};
```

`BridgeIntent` 不是本 PR 要加入的 wire 类型；它是未来客户端内核和 adapter 之间的内部
边界。协议字段、版本和服务端授权要在后续跨仓 PR 中单独定义。

## 5. 消息状态机

```text
                  ┌──────────────┐
                  │   CREATED    │
                  └──────┬───────┘
                         │ activity / first content
                  ┌──────▼───────┐
                  │    RUNNING   │
                  └──────┬───────┘
                         │ send/update process carrier
                  ┌──────▼───────┐
                  │ CARRIER_LIVE │──────┐
                  └──────┬───────┘      │ cancel / fail / NO_REPLY
                         │ terminal     │
                  ┌──────▼───────┐      │
                  │ FINAL_INTENT │      │
                  └──────┬───────┘      │
                         │ fresh send  │
                  ┌────
```