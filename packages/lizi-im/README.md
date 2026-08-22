# @cindy/im

纯 Node IM 传输包。为希望暴露 IM 机器人接口的主机（host）提供 `BaseIM` 抽象和开箱即用的渠道实现（目前为飞书）——在飞书中聊天，工作发生在主机中。

该包**与任何应用程序完全解耦**——主机代码注入该包自身无法提供的所有能力（加密存储、IPC 桥接、文件路径、可选日志记录器）。设计目标是从本仓库中独立出来，作为独立的 npm 包发布。

## 设计约束

- **零运行时依赖**，仅包含 IM 协议实际所需的依赖：
  - `@larksuiteoapi/node-sdk`（飞书 WebSocket + REST）——仅此一个
- **不包含** `electron`、`@cindy/maker-core`、`@cindy/mcps`、`drizzle-orm`、`@paralleldrive/cuid2` 或工作空间内部包
- **不导入** `apps/**`（由源码目录边界强制约束；通过 ESLint 对 `electron` 的 `no-restricted-imports` 规则验证）
- **纯传输/连接层**——不感知代理、会话、名片、模型选择或数据库。主机拥有所有这些能力，并使用该包的外发 API 驱动飞书。

## 架构

```
              ┌──────────────────────────┐
              │  你的主机（apps/desktop    │
              │   或任何其他 Node 应用）    │
              └────────────┬─────────────┘
        IMHost 适配器 │    │  BaseIM API（init / dispose /
        （主机 → 包）  │    │   onMessage / sendText / ...）
                       ▼   ▼
               ┌──────────────────┐
               │     @cindy/im      │
               │  ┌────────────┐  │
               │  │  BaseIM    │  │   抽象类
               │  └─────▲──────┘  │
               │  ┌─────┴──────┐  │
               │  │ FeishuIM   │  │   具体实现（使用 @larksuiteoapi/node-sdk）
               │  └────────────┘  │
               └──────────────────┘
```

## 快速开始

```ts
import { createIM, createFeishuIM, type IMHost } from '@cindy/im';

// 1. 构建 IMHost —— 这是该包依赖的唯一接口。
const host: IMHost = {
  paths: {
    feishuMediaDir: '/path/to/your/app/data/feishu-media',
  },
  secrets: {
    isAvailable: () => true,
    write(name, plaintext) { /* 持久化加密 */ return true; },
    read(name) { /* 返回解密内容或 null */ return null; },
    remove(name) { /* 删除 */ },
  },
  ipc: {
    handle(channel, handler) { /* 绑定 invoke 处理器——用于设置界面 */ },
    broadcast(channel, payload) { /* 推送到所有 UI 窗口 */ },
  },
  // 可选：
  // createLogger(scope) { return /* 你的日志记录器 */; },
};

// 2. 实例化渠道并聚合。
const feishuIm = createFeishuIM(host);
const im = createIM([feishuIm]);

// 3. 生命周期。
im.registerIpc();           // 同步——注册设置界面处理器
await im.init();            // 异步——若已保存凭据则自动连接
// ... 应用运行中 ...
await im.dispose();         // 异步——优雅关闭（发送离线通知）

// 4. 订阅入站消息和卡片按钮点击。
feishuIm.onMessage((event) => {
  // event.text、event.attachments、event.unsupported、event.senderId、event.messageId、...
  // 决定如何处理——调用 LLM、运行工具等。
});
feishuIm.onCardAction((event) => {
  // event.buttonId、event.payload——你的卡片 Schema
});

// 5. 外发——随时向用户发送，不仅限于回复。
await feishuIm.sendText(openId, 'Hello!');
const handle = await feishuIm.startStreamingText(openId);
handle.append('流式传输块');
await handle.finalize('带 **markdown** 的最终文本');
await feishuIm.sendInteractiveCard(openId, {
  title: '选择一个',
  body: '正文文本',
  buttons: [{ id: 'a', label: 'A' }, { id: 'b', label: 'B' }],
});
await feishuIm.sendFile(openId, '/abs/path/to/file.pdf');

// 6. 白名单——处理入站消息所必需（仅限点对点）。
feishuIm.setAllowedSenderOpenIds(['ou_xxxx...']);
```

## 公共 API

完整的导出接口请参阅 `src/index.ts`。要点如下：

| 导出 | 用途 |
|---|---|
| `BaseIM` | 抽象基类——扩展以添加新渠道 |
| `createIM(channels)` | 聚合门面，提供 `init / dispose / registerIpc` |
| `createFeishuIM(host)` | 具体的飞书渠道 |
| `FeishuIM` | 类——暴露入站订阅和外发 API |
| `IMHost` | 主机必须提供的能力 |
| `IMMessageEvent` | 入站消息载荷（文本 + 附件 + 不支持类型 + 确认 ID） |
| `IMCardActionEvent` | 入站卡片按钮点击 |
| `IMStatus` | 连接状态联合类型 |
| `InteractiveCardSpec` | 外发卡片规格 |
| `StreamingTextHandle` | 限速流式文本句柄 |

## 渠道特定行为（飞书）

- **仅限点对点。** 群聊/话题聊天在 WebSocket 层被丢弃。外发 API 接受单个 `openId: string`（无 chat_id）。
- **需要白名单。** 若 `setAllowedSenderOpenIds([])` 为空，所有入站消息将被静默丢弃。由调用方（主机）决定允许谁——通常是当前登录的飞书用户。
- **流式文本中的 `xdt-image://` / `xdt-file://` 重写。** `startStreamingText` 句柄的 `finalize` 步骤会将任何 `xdt-image://` 引用上传到飞书（基于 mediaDir 解析）并在最终卡片中内联为 `img` 元素；`xdt-file://` 引用作为单独的文件消息发送并从卡片正文中移除。流式传输中途的补丁显示友好的占位符。
- **冲突检测。** 同一 App ID 的多设备占用通过 WebSocket 重连启发式算法（旧版 SDK 行为）检测，并以 `'feishuBot:conflict'` 推送形式呈现给 `host.ipc.broadcast` 调用方。
- **生命周期公告。** 连接/优雅断开时，向白名单中的第一个 open_id 发送"在线"/"离线"文本（尽力而为，停止时超时 1.5 秒）。

## 渲染器 / UI 兼容性

`registerIpc()` 注册的 IPC 通道名称是稳定字符串（`feishuBot:get-state`、`feishuBot:save`、`feishuBot:clear`、`feishuBot:status-change`、`feishuBot:conflict`），因此针对旧版 "feishuBot" 命名空间的现有渲染器代码无需修改即可工作。主机的 `IMHost.ipc` 适配器是唯一需要将这些桥接到主机 UI 传输层的部分。

## 测试

```bash
pnpm -F @cindy/im run build   # tsc --noEmit
pnpm -F @cindy/im run lint    # eslint（no-restricted-imports 强制禁止 electron）
pnpm -F @cindy/im run test    # vitest
```

## 添加新渠道

1. 创建 `src/{channel}/` 目录，其中 `index.ts` 暴露一个继承 `BaseIM` 的类
2. 添加 `createXxxIM(host)` 工厂函数
3. 从 `src/index.ts` 重新导出两者
4. 渠道实现拥有完全的自由——自己的依赖、自己的入站/外发原语、自己的卡片格式——只要遵守 `BaseIM` 生命周期契约，并将任何新的主机需求添加到 `IMHost` 接口即可