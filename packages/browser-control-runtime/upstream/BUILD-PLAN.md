# Browser Control Runtime — Vendoring 构建计划

> **历史文档（仅供溯源）。** Vendoring 已交付，日常维护见 `MAINTAINING.md`，现状见 `STATUS.md`。
> 本文保留最初的提取规划，用于理解 `_generated/` 的来历与 `sync.mjs` 的设计意图。

针对完整提取的权威计划（跨会话持久有效）。基于对上游提交
`b972feb3f791ed38dafc27c1961dc87f2e30b210`（extensions/browser）及已发布的
`@openclaw/fs-safe@0.4.0` npm 包进行的依赖闭包分析生成。

面向产品的中立性：上游项目名称仅出现在此 `upstream/` 元数据目录和同步脚本中。生成的代码保留承载关键逻辑的内部标识符，但绝不会向用户展示；包名、MCP 工具名和设置 ID 保持中立（`browser` / `cindy_browser`）。

## 目录结构

```
packages/browser-control-runtime/
  src/
    _generated/
      extension/      <- 133 个浏览器核心文件（extensions/browser/src/**），逐字复制
      packages/       <- vendored 上游工作区包（源码来自 tarball）
        net-policy/
        normalization-core/
      vendor/
        fs-safe/      <- @openclaw/fs-safe 0.4.0 dist（89 个文件，零依赖 ESM）
    shim/             <- 手写的插件 SDK 子路径替换实现
    runtime.ts        <- createBrowserControlRuntime(): BrowserControlRuntime
    host.ts           <- 宿主注入契约（config/logger/security/media）
```

`_generated/` 下的所有内容均为生成产物——请勿编辑。重新运行 `pnpm sync:browser-runtime`。

## 源码来源分类（实测数据，非估算）

| 分类 | 数量 | 来源 | 方式 |
|---|---|---|---|
| 浏览器核心 | 133 | extensions/browser/src（GitHub tarball @ 指定提交） | 逐字复制，将 plugin-sdk 导入重写为 shim |
| net-policy | 1+ | packages/net-policy/src（tarball，未发布） | 复制源码 |
| normalization-core | 4 | packages/normalization-core/src（tarball，未发布） | 复制源码 |
| fs-safe | 89 | @openclaw/fs-safe@0.4.0 npm dist（不在 tarball 中） | 复制 dist，在 lock 文件中锁定版本 |
| SSRF/安全叶子节点 | ~14 | src/infra/net/**、src/security/**（tarball） | 复制源码（忠实保留——安全核心能力） |

安全闭包实际需要的外部 npm 包：`ipaddr.js`、`undici`、
`zod`（均已在 monorepo 中）。`fs-safe` 为零依赖。

## Shim 表面（手写，src/shim/*）

叶子工具子路径——从 tarball 源码忠实移植：
- `string-coerce-runtime`、`number-runtime`、`text-utility-runtime` → 重新导出 vendored normalization-core
- `browser-config`（parseBrowserHttpUrl/redactCdpUrl/movePathToTrash）
- `security-runtime` → 重新导出 vendored fs-safe + SSRF 叶子节点 + secret-equal + external-content
- `ssrf-runtime` / `ssrf-runtime-internal` → 重新导出 vendored SSRF 叶子节点
- `temp-path`、`json-store`、`process-runtime`、`error-runtime`、`routing`

宿主耦合子路径——最小正确实现 / 宿主注入：
- `logging-core`（createSubsystemLogger/redactSensitiveText/redactToolPayloadText）→ console 日志器 + 忠实的 redact 子集；不引入 54 配置类型的 tslog 依赖图
- `config-contracts` / `runtime-config-snapshot` / `config-mutation` / `plugin-config-runtime` / `core` → 内存配置提供器，从宿主注入初始数据（浏览器配置、SSRF 策略、端口）
- `media-runtime` / `media-mime` / `media-understanding-runtime` → 宿主注入的图像缩放 + describeImageFile 回调（Cindy 图像理解基础设施）；安全默认值 = 明确报错
- `gateway-runtime` → 分发路径仅需 isLoopbackHost / resolveGatewayAuth / ensureGatewayStartupAuth；仅回环地址，使用宿主令牌
- `cli-runtime` / `channel-actions` / `agent-harness-runtime` / `plugin-entry` / `plugin-runtime` / `setup-tools` / `runtime-env` → 薄桩实现（分发路径仅涉及 formatCliCommand、jsonResult、图像辅助函数）

## 入口点

`src/_generated/extension/src/browser/local-dispatch.runtime.ts`
→ `dispatchBrowserControlRequest(req)` 驱动进程内路由分发器。
`runtime.ts` 将其封装在 `BrowserControlRuntime.call(request)` 契约之后。

## 为何不 vendoring SDK 本身

沿 plugin-sdk 的 barrel 导出传递追踪会触及约 3957 个单体仓库文件。这些 barrel 导出会扇出到整个应用（agents、channels、gateway）。我们仅 vendoring 133 个文件的浏览器核心 + 有界的安全叶子节点，其余部分通过 shim 实现。