# 两个 Telegram bot 的能力台账

Cindy 有两个 Telegram bot，用户看到的是同一个产品：

- **官方 bot**——服务端 `telegram-hook-server` 持有 Telegram 连接，桌面端通过 device-link 帧（`turn.progress` / `turn.end` / `msg.op`）驱动它。桌面侧那一半在 `apps/desktop/src/main/hook-control/**`。
- **个人 bot**——桌面端自己持有 Telegram 连接（`packages/lizi-im/src/telegram/**`），在进程内渲染，状态落本地库。业务侧在 `apps/desktop/src/main/im/**`。

两者是**两套架构**，不是同一份代码的两个开关。所以"统一"不等于"让两条代码路径长得一样"，而是：**能同源的同源；不能同源的，差异必须写在这张表里、有裁决、有理由。**

> 这张表存在的理由与 `botCommands.ts` 注册表一样：把「散落两个仓、谁也不知道差在哪」变成「表里显式登记」。**新增或修改任一 bot 的用户可见行为时，必须同步更新本表对应行**；差异可以有，但不能没人说得清。
>
> 写这张表时的教训：**不要读模块注释就下结论，要读那条路径最后真正交出去的是什么。** 初版据此把「终稿保留过程区」写成了两侧差异，实际两侧**成功收口**时都只交正文。（失败收口两侧确有差异——个人侧留「过程区 + 正文 + 错误」，官方侧走独立错误字段，见第二节末行。成功与失败必须分开看。）

判定口径三档：

| 档 | 含义 |
|---|---|
| **同源** | 字面上同一份代码/同一份数据在两侧生效，不可能漂移 |
| **有意不同** | 已经裁决过的差异，**不要去"统一"它**，动它要先推翻裁决 |
| **缺口** | 该有而没有，或两边各写一套。这一列是待办 |

---

## 一、已同源

> **这一节只放两侧真的跑同一份代码/同一份数据的东西。** 一旦列进来，维护者就会跳过双路核对——所以「个人侧独有」「官方侧独有」的能力不能放这里，哪怕它在共享目录下。
>
> 还要写清同源的**是哪一段**，并且**指到真正干活的那个文件**。群历史检索被拆成三行就是这个原因：真正跑查询的是 `groupHistorySearch.ts`、管这一轮能不能查的是 `groupHistoryAccess.ts`、把两者串起来的是 MCP 入口——只写一个模块名会把维护者引到只负责权限租约的那一半，漏查真正的检索实现。而「这一轮有没有权限查、能查多大范围」那一步两侧各写各的，私聊上给出的答案还不同（第四节 2e）。

| 能力 | 单一真相源 | 共享到什么程度 |
|---|---|---|
| 过程区与正文的**文本合成** | `im/shared/turnPresenter.ts` + `turnActivity.ts` | 过程区怎么排（工具步骤、思考步骤、耗时行）、过程区与正文怎么拼（`composeProgressView`）。**正文累积不算**——见第三节：`createTurnPresenter` 按 `mode` 实例化两个独立引擎，累积、消息投影、`finalText()` 判据都不同，改一个引擎不影响另一个 |
| 群历史**检索实现** | `im/shared/groupHistorySearch.ts` | 真正执行查询的就是这一份：FTS（`hook_group_messages_fts` 的 MATCH）+ 中文 LIKE 兜底，**lane 条件写在 SQL 里**（MATCH 与 LIKE 用完全相同的 lane 条件，调用方没法先全局搜再事后过滤），加上结果映射（snippet / score / source）与上限（默认 8 条、最多 20 条、query 256 字）。两侧共用 |
| 群历史检索的**逐 turn 授权租约** | `im/shared/groupHistoryAccess.ts` | 租约机制共用：`beginGroupHistoryAccess` 在 provider 真正开始这一轮之前登记作用域，终态 / 重排 / 失败时释放；`sessionInstanceId` 挡住「同一业务 session 重建后，旧 MCP 请求借用新实例权限」。**但产生 scope 的那一步各写各的**——个人是 `im/telegram/adapter.ts` 的 `groupHistoryAccessFor`，官方是 `hook-control/groupHistoryScope.ts` 的 `groupHistoryAccessForExternalKey`；群轮次两侧同为 lane-only，**私聊上给的不一样**，见第四节 2e。完整调用链：MCP 入口 `mcp-integrations/groupHistoryMcpServer.ts` 先 `readGroupHistoryAccess` 拿租约（拿不到直接 `NO_ACTIVE_TELEGRAM_SCOPE` 拒绝），再 `resolveTargetLane(scope, lane)` 定位，最后才调 `searchGroupHistory` |
| 交互卡的**语义层**（`ask_user_question` / `plan_review` / 权限确认） | `im/shared/interactionCardModel.ts` | 选项集与决策模型两侧真跑同一份：至多 6 个选项（`MAX_OPTIONS`）、multiSelect 降级单选、只渲染第一问、plan 正文截断 1500（`MAX_PLAN_LEN`）、按钮文案的**产品级**上限 30（`BTN_LABEL_MAX`）、无选项降级时唯一按钮「继续」，以及 buttonId → 决策对象的构造与 header/body 拆分。**2026-08-19 起该模块另有一份全量视图**（顶层 `questions` 数组 + `needsAskMultiCard` 判定），只被支持卡片原地更新（`updateInteractiveCard`）的个人 IM 渠道消费——目前**仅飞书个人 bot**（`ui.cards.ask.multi` 打勾卡，多题/真多选，见 `cardBuilders.buildAskMultiCard`）；**本表对照的两个 Telegram bot 都不消费它**，上列 v1 规则对两侧与未提供 multi 文案的渠道仍逐字成立，看到飞书行为不同不要当漂移报官方那条链路由 `hook-control/interactions.ts` 直接 import 本模块（对 `@cindy/maker-core` 刻意只做 type-only 依赖，免得 hook 链路在运行期加载整个 barrel）。**渲染不在这里**——按钮文案来源、标题格式、省略号样式、尺寸上限都是各自渲染侧的事，见第三节 |
| **计划对账注入**（plan reconcile） | `maker-ipc/planReconcile.ts`（`summarizeOpenPlan` + `buildPlanReconcileNote`） | 两侧在真实用户轮次发送前查询未收口计划并前置对账说明。官方侧：`hook-control/session-runner.ts`（仅 `req.source?.im` 存在时注入,自动派发不注入）；个人侧：`im/shared/turnRunner.ts`。两侧共用同一份查询与文案构造,差异仅在"什么算普通用户轮次"的判定各走各的来源分类 |
| 精确 `pi install/update/remove` 的**受管扩展路由与回执** | `packages/maker-core/src/agents/pi/index.ts` 的 `routeManagedPackageCommand` / `piManagedPackageVisibleReceipt`，授权落点为 `maker-host/pi-managed-package-mutation.ts` | 两侧都把 Main 保存的原始频道文字经 `MAIN_OWNED_SEND_CONTEXT` 交给同一份整消息解析、相对路径解析、受管 store 变更和有界 JSON 回执：官方侧由 `hook-control/session-runner.ts` 优先传服务端 `source.userText`（旧服务端才回退已装饰 prompt），个人侧由 `im/shared/turnRunner.ts` 传 adapter 的 `item.text`。**确认有意不同但判据同源**：个人 bot 的已认证 IM 原文就是本次安装／更新／卸载授权，不再二次弹窗；官方 bot 的 hook 原文仍走 `local-desktop-command`，必须在 Desktop 原生确认框批准，确认面不可用或取消即失败关闭。两侧成功／失败都由同一份 Core 回执进入任务消息与模型上下文，宿主绝对路径被剥离，兼容性、资源风险与「仅新任务快照生效」说明一致；可执行 Extension 安装后仍保持未启用，必须在设置页按指纹另行批准。消息最终落在 Telegram 的载体形态继续遵守第二节既有生命周期差异 |
| 群消息本地库的**保留策略** | `im/shared/groupWindowCore.ts` | 上限数值（每命名空间 1 GiB 正文 + 500 万行安全阀）、回收低水位（0.9）与回收实现都在这一处；两侧把同一份 `DEFAULT_GROUP_WINDOW_RETENTION` 传进同一个 `recordGroupWindowEntry`。**额度靠 provider 命名空间隔离**：官方 `telegram:<principalId>`、个人 `telegram-personal:<botId>`，统计与回收都按 provider 过滤，两个账号各算各的、消息不串。一个边界要记住：两侧各持有一份 `{ ...DEFAULT }` **可变副本**（为的是测试能用小阈值把回收逼出来），所以共享的是"模块初始化时的那组数字"，运行期改一侧不会传导到另一侧 |
| **上游过载 / 限流自动重试进度** | `im/shared/turnRetryNotice.ts`（`turnRetryNotice`） | 非终止 error 翻成渠道进度的**文案与判定**两侧真跑这一份：只认过载（`(auto-retry N/M)` + `upstream-overload` / 529）、终态 429 外层重投、以及 Auto 档审阅器不可用。Pi / Claude / Codex 过载重试都走「模型服务繁忙，正在自动重试（N/M）」；其它非终止 error（普通 5xx、未分类供应商抖动）保持静默。个人侧 `turnRunner`、官方侧 `turnPresenter` 都读这一份。**载体怎么发**不在这里——个人是过程消息原地编辑，官方私聊是草稿，见第三节 |

### 放在共享目录、但**只有一侧消费**的

| 东西 | 实际情况 |
|---|---|
| `im/shared/channelToolPolicy.ts` 的 `channelForceConfirmToolCall` | 只被个人 Telegram / 微信 / 钉钉的权限策略引用。**官方 bot 不挂**——见第三节的裁决。放在 `shared/` 下是因为个人侧三个渠道共用，不代表两个 bot 共用 |
| `packages/lizi-im/src/telegram/presentationCapabilities.ts` | 只导出并由**个人 driver** 消费 `TELEGRAM_PERSONAL_CAPABILITIES`，没有官方 bot 共用的契约数据。它的作用是把车道差异写在一处，不是让两侧取同一份值。官方侧同名策略在另一个仓各写一套——具体到链接预览见第四节 2c |
| `PresenterPolicy.intermediateMaxRenderedChars`（长度上限） | **只有官方那条路消费**（`createProgressEmitter`）。个人侧用自己的私有常量 `INTERMEDIATE_EDIT_LIMIT = 3800`（`streamingText.ts`）判断何时停止编辑。**改共享的长度策略只会改到官方 bot**——两处独立维护，改一处必须核对另一处 |
| `PresenterPolicy.intermediateThrottleMs`（节流间隔） | 个人路径是**双层节流**：`turnRunner` 的 `CARD_PATCH_THROTTLE_MS` 确实读共享值，但真正出站的 `streamingText.ts` 还有一份写死的 `TELEGRAM_UPDATE_THROTTLE_MS = 1500`（注释称「双层节流冗余但无害」）。**改共享值只改得动 runner 那层，driver 那层不跟**——所以这个值也不是同源，改它要连 driver 的常量一起核对 |
| `im/shared/botCommands.ts` 的**官方那一半** | 官方 bot 的命令**仍由服务端 `TELEGRAM_COMMANDS` 下发**，本表对官方侧是「声明性镜像、不接线」。测试只用内联清单核对镜像，**服务端改了命令这边完全可能不同步**。个人侧那一半是真的单一真相源（菜单与分发直接读它）；官方那一半是跨仓镜像，**改命令要两个仓一起核对** |

进度帧去重的三槽基线（`shouldEmitProgressFrame` / `createProgressEmitter`）同理：只在注入 `onProgress` 时启用，也就是**只有官方那条路在用**，个人侧不消费。

## 二、消息生命周期——按阶段逐格对照

`turnPresenter` 统一的是"这一刻该显示什么"这段文本。"这段文本发给谁、放在哪、什么时候变、最后落在哪"由各消费方自己负责（模块注释明写：收口不在 presenter 里）。

| 阶段 | 个人 bot | 官方 bot |
|---|---|---|
| 首帧 | 有真实内容（含工具步骤）就建一条**真实消息**；空内容不建（惰性占位） | 快照进 `turn.progress` 帧发给服务端 |
| 过程中 | **第一帧真实内容用 `sendMessage` 建消息（这一条会推送），之后持续 `editMessageText` 覆盖（编辑不推送）**，用户看着它长大：过程区在上、正文在下，正文是整轮累计内容 | **私聊**：进 Telegram **草稿**（`sendDraft`）——在输入框那个位置，**不在消息流里**；**群**：一条进度消息，`editMessageText` 覆盖。桌面端对 Telegram 专门用 `progressBodyMode: 'whole'`，在 3800 字单帧上限内同样累计展示整轮正文；超过后退回当前 assistant 消息，避免头部截断把最新答案藏掉。Slack / X 仍只展示当前 assistant 消息 |
| `done` 交接 | 最后一帧由个人 driver 自己定稿 | 桌面端在 `turn.end` 之前先 `flushProgress()`，跳过尚余的 1.5 秒尾沿节流，把最新安全快照放进既有进度载体；这是防止 observer teardown 吞掉最后一帧的客户端兜底，**不等于**服务端终稿已发布成功 |
| 终稿内容（**成功收口**） | **只有正文**（`composeStreamingView` 在 `turn.done` 时直接 `return body`，不再合成过程区） | **只有正文**（`presenter.finalText()` 取 body 引擎的缓冲，不经过 `composeProgressView`） |
| 终稿落在哪 | **永远新发一条独立消息**，落地后才尽力删掉停在过程态的旧载体——**删不掉就两条并存**。优先 `sendRichMessage`（表格/公式原生渲染、32768 上限免分段）；**Telegram 完整应答的任一 4xx** 都判为「这条 Rich 没落地」并回落新发 HTML——判据是**有没有拿到应答**而非错误码大小：404（方法缺失，另触发实例级熔断，后续不再试 Rich）、400（本条解析不过）、**429（`callSend` 按 `retry_after` 退避重试后仍限流；不熔断，下一轮照常试 Rich）**。抛错只留给拿不到应答的情况（网络中断、超时、5xx）——那时无法判断 Telegram 是否已接收，补发 HTML 可能造成两份答案。超长时第 2 段起逐段 `send`。**受管图片**让终稿跳过 Rich 直接走 HTML 新发，图片随后由 `uploadImages` 挂到新终稿上。过程载体从不承担答案，因此最后一次编辑撞 flood 不再丢终稿 | **私聊**：新发一条正文消息，草稿随之消失；**群**：编辑那条进度消息 |
| **失败收口**（普通轮次） | **过程区保留**：错误路径不置 `turn.done`，`composeStreamingView` 仍走运行中合成——卡片定稿成「过程区 + 正文 + ❌ 错误：…」，用户能看到失败前干到了哪一步 | 终稿正文为**空**，错误信息走独立的 `errorMessage` 字段，由服务端按语言渲染成「任务失败：…」——**不带过程区** |
| **失败收口**（群开了 `always` 的 **ambient 轮次**） | **和普通轮次一样**照吐 `❌ 错误：…`——`turnRunner` 不认识 ambient。惰性占位这时会被真建出来，群里凭空多一条错误消息，而这一轮本来连话都不打算说。**这是缺口 2f，不是裁决** | **静默**：不发失败通知（`finalFailureNoticeSent !== true && !entry.ambient`），删掉过程消息、记一句「completed silently」。删不掉时标 `retainAmbientCleanup` 留给下一拍重试 |

**成功收口的终稿两侧都只有正文**——这一点没有差异，不要登记成缺口。但它带两条限定，少写一条就会变成假不变量：

- **只对成功成立**。失败收口两侧形态不同（上表最后两行），个人 bot 保留故障现场、官方 bot 只给一句错误。改收口逻辑时不要拿「终稿只有正文」去删失败路径的过程信息——那是用户排障的唯一线索。
- **失败收口本身还要分普通轮次与 ambient 轮次**，两侧的差别正好反过来：普通轮次是「个人留现场 / 官方一句错误」，ambient 轮次是「个人照样吐错误 / 官方全静默」。所以上表把失败拆成两行——一个无条件的「失败怎么收口」结论会同时说错其中一半（缺口 2f）。
- **「过程区消失」也不是无条件的**。个人 bot 的终稿永远是新消息，旧的过程消息是**尽力删**——`deleteMessage` 失败被吞掉（权限、已被删、API 报错），于是聊天里会同时