# 开发规则

此处存放 Cindy 客户端仓库的工程约束、实现规则、开发流程与验证方法。

## 收录标准

- 规则需说明“代码如何实现、如何验证，或哪些技术操作被禁止”。
- 每条规则应明确触发条件、必须执行的操作、禁止事项、验证方法及例外情形。
- 仅适用于单个目录或模块的规则，优先放置于对应目录的嵌套 `AGENTS.md`；需跨目录复用或需较长解释的规则，则收录于此。
- 凡可由 lint、测试、类型检查或脚本强制执行的规范，必须同步落实为自动化检查，不得仅依赖 Agent 阅读文字。

## 当前文档

- [`repo-map.md`](repo-map.md)：仓库地图——apps、packages 及顶层目录的定位导航。
- [`environment-setup.md`](environment-setup.md)：公共开发环境、依赖安装与 submodule 准备。
- [`desktop-development.md`](desktop-development.md)：Desktop 的 Agent 安全启动入口与分层验证命令。
- [`electron-security-and-process-boundaries.md`](electron-security-and-process-boundaries.md)：Electron 进程职责、Renderer 信任模型、BrowserWindow、独立辅助窗口统一生命周期、preload、IPC 与远程内容安全边界。
- [`credentials-and-local-storage.md`](credentials-and-local-storage.md)：凭证不入仓、用户持久数据、临时文件与测试目录的安全边界。
- [`media-storage-and-protocols.md`](media-storage-and-protocols.md)：Desktop 媒体总仓、`cindy-media://` 协议、引用生命周期与历史兼容边界。
- [`database-and-migrations.md`](database-and-migrations.md)：Desktop SQLite schema、append-only migration、companion script、隔离运行与异步数据库访问规则。
- [`mobile-development.md`](mobile-development.md)：Mobile 的模拟器开发、分层验证与专项文档入口。
- [`orca-team-architecture.md`](orca-team-architecture.md)：Orca 多 Agent 协同架构与运行时约束。
- [`maker-core-and-agent-behavior.md`](maker-core-and-agent-behavior.md)：`packages/maker-core` 的 Agent 能力归属、代码优先确定性、缓存率／性能／准确性指标不可回退，以及 system prompt 改动门禁。
- [`plugin-security-and-authoring.md`](plugin-security-and-authoring.md)：插件（`.cindy`）运行时沙箱与进程隔离、权限即授权边界、网络／凭证／资源交接、存量插件向下兼容红线（升级不得要求用户重装或重新确认），以及作者契约与编写手册（`FORGE_GUIDE`）同步。
- [`cindy-updater.md`](cindy-updater.md)：客户端自动更新链路的 owner 确认门禁与高风险约束。
- [`engineering-conventions.md`](engineering-conventions.md)：统一日志、IPC 错误协议（`throwIpcError`）、main 侧默认带测试、macOS／Windows 双端兼容，以及 UI 文案的 i18n 落地与 `pnpm check:i18n` 门禁。
- [`log-upload-and-redaction.md`](log-upload-and-redaction.md)：客户端日志采集／脱敏／上报的三条不变量（记录边界、白名单方向 deny-by-default、标记代次 + 原子清除）、四层收窄管道、授权闸、区域绑定与崩溃时序。
- [`protocol-compatibility.md`](protocol-compatibility.md)：两仓本地协议实现、device-link relay 层、插件来源与 wire protocol 兼容。
- [`architecture-invariants.md`](architecture-invariants.md)：package 与 render／main 解耦、main 进程静态依赖，以及主界面布局树不变量。
- [`configuration-and-overrides.md`](configuration-and-overrides.md)：配置可见性分层、默认值与用户 override 分离、默认值迁移与恢复默认语义。
- [`remote-and-mobile-adaptation.md`](remote-and-mobile-adaptation.md)：SSH 远程工作区、device-link allowlist、手机版入口与功能类 PR 的远程／手机三选一门禁。
- [`development-workflow.md`](development-workflow.md)：worktree dogfooding 会话契约、提 PR 与直推 `main` 门禁，以及 Review P0／P1／P2 严重度口径。