# 视觉桥（agent-vision-toolkit 式）集成方案书

> **状态**：已定稿（2026-08-11）。本文件是实现的唯一锚点，任何改动先改这里再改代码。
> **分支**：`feat/vision-toolkit-text-models`
> **范围**：让纯文本模型（如 deepseek / glm 等不支持视觉的模型）获得看图能力——用外部多模态
> API 把图转成文字描述，再让纯文本模型基于文字继续工作（image-to-text 层，vision token 不
> 直接交给文本模型）。

---

## 一、目标与硬约束

**目标**：给"包括但不限于 deepseek"的纯文本模型补视觉能力。

**硬约束**（必须满足，不得破坏）：
1. **默认不开启**——只有用户显式启用才生效；
2. **不影响正常模型使用**——纯文本模型正常流程零干扰，视觉模型完全不受影响；
3. **三层全做**：B（session 层贴图主动调视觉）+ A（proxy 透明替换兜底）+ C（pi 工具模式增强），
   pi-only 的 C 也做。

---

## 二、现状与关键事实（已确证，改动时不得偏离）

| 环节 | 事实 | 代码锚点 |
|---|---|---|
| 贴图归一化 | 图片统一为 `{ type:'image', path: absPath }` | `packages/maker-core/src/types/common.ts:46-53` |
| 消息入口 | `session.send()` 组装 `UserMessage` 后交给 agent（视觉桥在 `Session.send` 内、dispatch 前调用） | `packages/maker-core/src/session.ts` |
| cc 转图 | `toClaudeSdkContent` → `@"path"` mention → SDK 读文件 | `claude-code/index.ts:416` |
| codex 转图 | `toAppServerInput` → `{type:'localImage',path}` / `{type:'image',url}` | `codex/index.ts:1314,1361,1366` |
| pi 转图 | `buildPiPrompt` → base64 `PiPromptImage[]` | `pi/index.ts:301` |
| 多模态判定 | `supportsImageInput` 派生链只消费到 pi（cc/codex 不用） | `apps/desktop/.../catalog-to-descriptors.ts:89` |
| pi 前置拒绝 | `assertImageInputSupported` 抛 `PiImageInputUnsupportedError` | `pi/index.ts`（`assertImageInputSupported`） |
| cc/pi 代理装配 | `anthropic-compat-proxy`（`transformRequest` 链，视觉桥在 `stripNonAnthropicFields` 前） | `anthropic-compat-proxy-host.ts` |
| codex 代理装配 | 同包独立实例，transform 链 `createTransformRequestChain` | `codex-proxy-host.ts`（`createTransformRequestChain`） |
| 响应侧 | **只读 observer，无响应 transform**（SSE 低延迟刻意不做） | `packages/anthropic-compat-proxy/src/types.ts:6-9` |
| 上游失败三形态 | 显式 400（recovery 可捕获）/ **静默丢弃**（glm 实测，无信号，只能主动判定）/ pi 发送前拒绝 | `transform.ts`（#794 strip handler 表）、`pi/index.ts` |

**请求路径覆盖矩阵**（决定视觉桥能覆盖哪些请求）：

| 请求来源 | wire 格式 | 走哪个 proxy | 过 transform 链？ | 视觉桥能否覆盖 |
|---|---|---|---|---|
| claude-code 子进程 | Anthropic Messages（`messages[]`） | cc 实例 | ✅ | ✅ A |
| pi 网关模型 | Anthropic Messages | cc 实例（pi 复用） | ✅ | ✅ A |
| codex 子进程（网关路线） | OpenAI Responses（`input[]`） | codex 实例 | ✅ | ✅ A |
| 订阅直连模型（chatgpt/ xai/） | Responses 经 localHandler 翻译 | 命中 `localHandler` | ❌ 绕过 | 本就是视觉模型，无需桥 |
| pi BYOM 原生 provider | pi 原生 | **直连用户端点，不过 proxy** | ❌ | ✅ B（session 层） |
| **ghost 工具结果图片** | MCP tool_result 纯文本（`cindy-media://` URL） | 不过 proxy（MCP 内建） | ❌ | ✅ **D（ghost 收口描述）** |
| WS 隧道流量 | socket 转发 | — | ❌ | 纯文本模型不走 WS，无影响 |
| 远程 SSH env 模式 | 远程直连 | — | ❌ | 低优先级，标注"远程需另行配置" |
| 标题 one-shot / 非 POST 控制面 | 直连 / 无 body | — | ❌ | 无图，无影响 |

**关键结论**：
- 层 B（session 层）是唯一能覆盖 **pi BYOM 原生直连**的层（A 的盲区）；
- 层 A 是唯一能看到 **tool_result 内嵌图片**的层（B 覆盖不了）；
- 层 C（工具模式）提供 B/A 结构性给不了的**主动多轮聚焦**（locate → crop → 细看）；
- **层 D（ghost 收口描述）**：工具结果里的 `cindy-media://` 图片是纯文本 URL，层 A/B 都扫不到
  （工具结果不是 image block、不经过 session hook）——纯文本模型只能看到 URL 文本，实测会**幻觉
  编造**图片内容。host 在 `ghost_call` 收口处读 blob 调视觉桥转成描述，附加为
  `xdt_media_descriptions`（可选字段，视觉桥关闭/无图/全失败时不出现，存量兼容）。

---

## 三、总体设计：三层互补

```
用户贴图 → [B] session 层主动调视觉（第一刀，侵入最小，覆盖三 agent + pi BYOM）
工具返回图 → [A] proxy 透明替换（兜底，覆盖 tool_result 内嵌 + 网关模型）
           ↘ [D] ghost 收口描述（工具结果 cindy-media:// URL → 文字描述）
模型主动追问 → [C] pi 工具模式（多轮聚焦，pi 成本最低）
```

| 层 | 触发时机 | 模型感知"有图" | 多轮聚焦 | 零干扰 | 侵入性 |
|---|---|---|---|---|---|
| B session 层主动调 | 用户贴图瞬间 | ✅ 收到"贴了图，已描述" | 弱（描述一次，可再追问） | ✅ 未启用原样透传 | 改 session 一处 |
| A proxy 透明替换 | 图片到上游 | ❌ 被动得描述 | ❌ 一次性 | ✅ 未启用 null 透传 | 改 proxy 双实例 |
| C 工具模式 | 模型决定 | ✅ 主动 | ✅ 强 | ✅ 工具不注入即不存在 | pi bridge 一处 |
| D ghost 收口描述 | 工具结果返回瞬间 | ✅ 收到"图已转文字描述" | 弱（描述一次） | ✅ 未注入/失败静默跳过 | host 一处（ghost.ts 收口） |

四层共用同一视觉通道后端（见第五节）。

---

## 四、各层详细设计

### 4.1 层 B：session 层贴图主动调视觉

**入口**：`Session.send` 内（msg 组装后、`agentHandle.send(msg)` 前，视觉桥调用点见
`packages/maker-core/src/session.ts`）。优先 session 层（覆盖所有 agent；不采用下沉到
`BaseAgent.send` 的方案）。

**流程**：
1. 扫描 `message.content` 的 `{type:'image'}` block（畸形非数组 content 安全返回空）；
2. 判定：视觉桥已开 + 当前模型 ∈ 启用组合 → 继续；否则原样透传；
3. 调视觉通道 `describeWithFallback(path, focusHint, signal)`，focusHint = 同消息 text 内容（比 proxy 层提取更准）；
4. 把 image block **替换**为描述 text block（前置来源标注「用户贴了一张图片，已由外部多模态模型转成文字描述」）。
   采用替换而非"前置注入保留图"的理由：纯文本模型拿到描述即直接可用、不会触发上游 400；
   层 A 对纯文本请求（无图）自然 no-op，不与层 B 重复描述。视觉模型本不该被配进启用组合
   （用户显式多选的是纯文本模型），故替换不损失其看图能力；
5. 视觉 API 失败/超时 → 单图失败用独立占位文本（「图片不可用，不要推测内容」）替换该图，其余成功图照常；
   仅当**全部图片都失败**时才回退无视觉桥状态，**原样把图透传** + 发出清晰提示（非终态事件）；
   调用方取消（Stop / detach / 外部 signal abort）**不计入失败/超时**：立即中止视觉请求并静默收口，
   不写占位、不发 note（取消是用户主动中断，不是视觉桥故障）；若调用链已因取消收口，则不会越过取消
   边界 dispatch 给 agent（hook 内部保留原 message 仅是返回值，不意味着把原图发给模型）；
   视觉通道自身 timeout 仍按失败处理；
6. **有界延迟（非无延迟）**：视觉调用在 `handle.send` 前 await，但有明确预算——单图超时 30s
   （视觉推理常需 10-30s，5s 太短易超时）、多图并发 2、单图主+fallback 最坏 ~60s、N 图约
   `ceil(N/2)*60s` 后 dispatch。取消（signal）可提前中止。这是「有超时的前置阻塞」，不是
   「不阻塞」；总预算（最大图片数/总耗时上限）为后续增强。

### 4.2 层 A：proxy 透明替换

**引擎改动**：`packages/anthropic-compat-proxy/src/types.ts` 的 `RequestTransform` 类型放宽为
允许返回 `Promise`（`RoutingTransform` 已 async 是先例）；`server.ts` `runTransforms`
循环改用 `isPromiseLike`（已存在）做 `isPromiseLike(r) ? await r : r`。
**现有同步 transform 全部兼容**；必须顺序 await（禁 `Promise.all`），保持
`repairToolExchangeAdjacency` → `dedupeDuplicateToolUseIds` 的顺序依赖（`transform.ts` 头注）。

**新 transform**：`createVisionBridgeTransform()`（`packages/anthropic-compat-proxy/src/`）：
- 短路：未启用 / 目标模型集不含该 model → 返回 `null`（字节透传，视觉模型零影响）；
- 双格式解析：
  - Anthropic Messages：`body.messages[].content[]` 的 `{type:'image'}` block（含 tool_result 内嵌）；
  - OpenAI Responses：`body.input[]` 的 `{type:'input_image', image_url}`；
- 命中 → 仅对 `http(s):` / `data:` 协议的 image_url 调视觉通道转文字 → **原地保序替换**为
  `input_text`/`text` 块；私有协议（`cindy-media://` / `xdt-image://` / `file://`）或裸本地路径
  是内部引用/本地文件，**不透传给第三方视觉后端**，直接降级为「图片不可用」占位 + warn；
- focus hint：最近 user 文本 / assistant 最后一段（参考 vision_proxy.py 的 `_last_paragraph`）；
- 失败 → 替换为「Image unavailable / 图片不可用」占位（对齐 vision_proxy.py），不 502，不阻塞。

**装配**（两实例都要加，视觉桥 transform 都在 `stripNonAnthropicFields` 前）：
- cc/pi 实例：`apps/desktop/src/main/maker-host/anthropic-compat-proxy-host.ts` 的 `transformRequest` 链（`buildVisionBridgeProxyTransform(log)`）；
- codex 实例：`apps/desktop/src/main/maker-host/codex-proxy-host.ts` 的 `createTransformRequestChain`。

### 4.3 层 C：pi 工具模式增强

**入口**：`packages/maker-core/src/agents/pi/cindy-bridge-source.ts`——已有一套 `pi.registerTool`
范式（`registerTool` 内置工具注册）、权限门、spawn 凭证剥离；本分支视觉工具 `vision` /
`vision-locate` 注册在 `cindy-bridge-source.ts`（`vision`/`vision-locate` 两个 `registerTool`）。

**注意**：cindy-bridge 跑在 **pi 子进程**，不能 import cindy 模块，只依赖 pi ExtensionAPI、
`fetch`、`node:fs`。所以层 C 的 vision 工具**在 bridge 内自包含 fetch 视觉 API**（复用与主机
侧 VisionChannel 相同的端点/凭证 env，由 host 通过 env 注入），不依赖主机侧通道对象。

**工具**（第一版实现）：
- `vision <path> [-q <focus>]`：glance 式问答，支持多轮（模型拿上次描述后可再 `-q` 追问细节）；
- `vision-locate <path> <target>` → 返回坐标（ground 式）。
- `vision-crop` **未实现**（设计参考 agent-vision-toolkit，后续增强）：`<path> <x1,y1,x2,y2> -o out.png` 放大细节复用。

**凭证**：视觉 API key 经 host 注入的 env（对齐 `CINDY_PI_API_KEY` 插值模式），进 spawn 边界的
`SECRET_ENV_NAMES`（`cindy-bridge-source.ts` 的 `CINDY_PI_SECRET_ENV_NAMES` 解析）剥离，模型拿不到。

### 4.4 层 D：ghost 工具结果图片描述

**背景（实测）**：模型调 ghost 插件工具（如 xd-feishu 读飞书群消息）返回的图片以
`cindy-media://blobs/<hash>.<ext>` **URL 文本**进 agent 上下文（MCP tool_result 是纯文本 JSON，
不是 image block）。层 A/B 都扫不到这种形态——纯文本模型（deepseek 等）只能看到 URL 文本，
读不到图，实测会**幻觉编造**图片内容（codex 会话编造了不存在的 not_anchor 判定结果）。

**入口**：`apps/desktop/src/main/mcp-integrations/ghost.ts` 的 `callGhostTool` 收口处
（`drainGhostCallMedia` 之后、返回前）。

**流程**：
1. `CindyGhostsHostDeps` 注入可选 `describeToolResultImage({imageUrl, sessionId, sessionInstanceId})`，
   host 侧（`maker-host/index.ts`）实现；
2. 收口处从 `producedMedia`（主机媒体账本）+ `result.result`（递归扫描 ≤8 层）收集
   `cindy-media://` URL，Set 去重，`blobStore.parseBlobUrl` + `mimeForExt` 过滤 image/*；
3. 限量并发（2）+ 整批总预算（60s）调视觉桥描述，单张失败静默跳过；预算是
   **完成门**：单个 budget timer 到点同时 abort 所有在飞请求（signal 透传到视觉通道
   fetch）+ resolve race；每张图 per-call 与预算 race（即使注入的 describeImage 不响应
   signal 且永不 settle，预算到期后该调用返回 null、worker 退出，不悬挂闭包）；
4. 成功描述附加为顶层 `xdt_media_descriptions: [{url, description}]`，随工具结果 JSON 透传给模型。

**判定**：`describeToolResultImage` 内——`getVisionBridgeController()` 未注入 / session 缺失 /
`session.instanceId !== sessionInstanceId`（旧实例）/ `shouldBridge(model)` 不命中 / blob 解析失败
→ 返回 null，静默跳过（每步 fail closed，工具调用永不阻塞）。

**兼容**：`xdt_media_descriptions` 是新增可选顶层字段，不在 `MEDIA_HOIST_KEYS`，渲染层/IM 出站
只消费固定媒体字段（`xdt_image_urls`/`xdt_video_urls`/`xdt_audio_tracks`/`xdt_media_produced`），
不会误消费。视觉桥关闭 / 无图 / 全失败时字段不存在，零影响。

---

## 五、视觉通道（三层公共后端）

### 5.1 复用现有 provider 体系（已确认）

视觉后端**不新增独立配置**，复用设置页里已配好的 provider。落地形态是
`apps/desktop/src/main/vision-bridge/vision-channel.ts` 暴露 **provider-scoped 解析函数**
`describeImageWithProvider(providerId, modelId, input, deps)`：**路由决策复用统一路由器**
（host 注入 `provider-route.resolveVisionBackendRoute`；与 agent 路由**决策同源**，最终对上游
的 model 还原结果一致——执行面不同：视觉桥直接发上游，代理链给子进程/代理 body 改 model）：
- **xd 特例**：客户端投影给 Codex 的模型（只声明 claude-code）走 Claude Messages 面，
  原生 codex 模型走 Responses 面——对齐 `providerRoutingForModel`。投影模型先剥 `codex/`
  + `[1m]` 路由前缀到裸 id（如 `codex/gpt-5.6-luna[1m]` → `gpt-5.