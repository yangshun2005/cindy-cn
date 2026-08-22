# @cindy/browser-control-runtime

面向 Cindy 的中立、进程内浏览器自动化运行时。它对外暴露一个小巧、稳定的契约（`BrowserControlRuntime`），供 `cindy_browser` MCP 和桌面宿主驱动——不依赖任何上游产品 API，也不会在产品代码中暴露上游产品名称。

## 目录结构

```
src/
  types.ts            公共契约：BrowserControlRuntime / Request / Result
  runtime.ts          createBrowserControlRuntime() —— 驱动契约背后的
                      内置进程内路由分发器
  unavailable.ts      一个安全的“未配置”运行时
  shim/               针对上游插件 SDK 的手写替代实现，
                      为内置核心导入提供支持（参见 shim-spec）
    _local/           对小型自包含上游辅助函数的忠实移植
                      （errors、ports、redact、tmp-dir、browser-cdp、text-utils）
  _generated/         自动生成 —— 请勿编辑。由 sync.mjs 生成：
    extension/        内置浏览器核心（extensions/browser/src/**）
    leaf/             内置 SSRF / 安全叶子闭包（src/infra、src/security）
    packages/         内置网络策略 + 规范化核心
    vendor/fs-safe/   内置 @openclaw/fs-safe 发行版（零依赖、版本固定）
  __tests__/          运行时 + SSRF 契约测试
upstream/
  MAINTAINING.md              维护者必读：三层架构 + 踩坑清单 + 上游同步 + 中性约束
  browser-runtime.lock.json   固定提交 + fs-safe 版本 + 内容哈希
  vendor-manifest.txt         133 个核心文件
  shim-spec.md                每个 shim 必须提供的精确命名导出表面
  STATUS.md                   交付现状
  BUILD-PLAN.md               历史设计笔记
```

> **改这块代码前先读 `upstream/MAINTAINING.md`** —— 它记录了集成层（desktop host + cindy_browser MCP）以及所有踩过的坑与不变量。

## 从上游更新

```
pnpm sync:browser-runtime --ref=<提交或分支>
pnpm --filter @cindy/browser-control-runtime build   # 暴露任何 shim 缺口
pnpm --filter @cindy/browser-control-runtime test     # 契约 + SSRF 防护
```

`_generated/**` 整体重新生成；切勿手动编辑。如需改变行为，请编辑 `sync.mjs`（供应商集合 / 导入重写）或手写的 `src/shim/*`。

## 安全说明

SSRF/路径包含的*决策逻辑*是内置的上游代码，而非重新实现。`src/shim/ssrf-runtime.ts` 仅重新实现了薄薄的 fetch *外壳*，用于组合那些内置原语（`isBlockedHostnameOrIp`、`resolvePinnedHostnameWithPolicy`、`createPinnedDispatcher`）。SSRF 契约测试断言真正的核心仍然会阻止云元数据和私有 IP，因此未来任何削弱这些能力的同步都会导致 CI 失败。