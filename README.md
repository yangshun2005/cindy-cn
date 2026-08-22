

> **[中文化] cindy**
>
> 此项目是 [makecindy/cindy](https://github.com/makecindy/cindy) 的中文翻译版本。
> - 原项目 Stars: 2249
> - 主语言: TypeScript
> - 许可证: Apache-2.0
> - 翻译日期: 2026-08-22
> - 原始 README: [README_en.md](README_en.md)
>
> 如有翻译不准确之处，欢迎提 Issue 或 PR。

---


---

<p align="center">
  <a href="https://cindy.app"><img src=".github/assets/hero-en.webp" width="100%" alt="CINDY — 搞定它。开箱即用的开源 AI 代理，在你的电脑上完成实际工作。"></a>
</p>

<p align="center">
  <strong>English</strong> · <a href="README.zh-CN.md">简体中文</a>
</p>

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-Apache--2.0-blue.svg" alt="License" /></a>
  <a href="https://github.com/makecindy/cindy/actions/workflows/ci.yml"><img src="https://github.com/makecindy/cindy/actions/workflows/ci.yml/badge.svg" alt="CI" /></a>
  <a href="https://nodejs.org"><img src="https://img.shields.io/badge/node-22.x-brightgreen.svg" alt="Node.js 22.x" /></a>
  <a href="https://pnpm.io"><img src="https://img.shields.io/badge/pnpm-10-orange.svg" alt="pnpm" /></a>
</p>

<p align="center">
  🌐 <a href="https://cindy.app">cindy.app</a> &nbsp;·&nbsp; ⬇️ <a href="https://cindy.app/download/">下载</a>
</p>


Cindy 是一个开箱即用的开源 AI 代理。她将多种工具框架（harness）、模型和工具整合到一个代理中，在你的项目和应用程序中完成实际工作。从第一天起就准备就绪，并随着时间推移由你塑造。

Cindy 在你自己的机器上本地运行，使用你的真实文件和已登录的应用程序。首批支持的工具框架是 **Claude Code** 和 **Codex** —— 更多正在添加中，一个原生工具框架也在开发中。模型和工具框架可以自由组合，并可在任务进行中切换，而你的工作区、记忆、技能和工具保持连续；一个任务甚至可以由不同工具框架 × 模型组合的代理进行规划、并行执行和审查。她可以驱动你的浏览器、电脑和手机，并从即时通讯和日程中接手工作。

本仓库是 Cindy 的开源**客户端** —— 桌面和移动应用程序及其共享包，组织为 pnpm monorepo。

该客户端免费使用，其源代码在 Apache-2.0 许可下开放。以你的方式使用模型：登录官方 Cindy 服务（透明扣除用量），授权你已经付费的 **Claude Code / Codex 编码计划** 并在 Cindy 内继续使用 —— 无需重复付费 —— 连接你自己的 API 密钥，或使用本地模型。
有关服务详情、[定价](https://cindy.app/#pricing)和[下载](https://cindy.app/download/)，请参阅 [cindy.app](https://cindy.app)。

## 由你塑造

开源不仅仅意味着可见 —— 更意味着可改变：

- **记忆** —— 纠正她一次，她从此就能做对，并在各工具框架间共享。
- **技能** —— 教授一次工作方式，随处复用；将它们传递给团队的功能正在开发中。
- **自动化** —— 重复性工作自行安排、自行运行、自行汇报。
- **MCP** —— 将你的内部工具和业务系统接入她的能力范围。
- **插件** —— 重塑功能、UI 和交互，通过开放市场共享 *(开发中)*。
- **源代码** —— 审计、Fork、扩展，并在 Apache-2.0 许可下回馈改进。

开箱即用，永不受限 —— 从 [`CONTRIBUTING.en.md`](CONTRIBUTING.en.md) 开始，与我们一起构建 Cindy。

## 本仓库内容

| 路径 | 描述 |
| --- | --- |
| `apps/desktop` | Electron 桌面客户端 |
| `apps/mobile` | Expo / React Native 移动客户端 |
| `packages/*` | 共享客户端能力（认证、设备链接、代理编排、模型提供商等） |
| `apps/*-bin` | 随桌面应用发布的工具二进制文件；均未提交 —— claude-code、codex 和 ripgrep 由 `pnpm install` 按平台下载，Android 平台工具二进制文件在 Windows 打包前获取（固定版本，sha256 验证） |

**不在此仓库中：** 后端服务位于单独的仓库中，不属于此 monorepo 的一部分。

| 模式 | 账户要求 | 可用性 |
| --- | --- | --- |
| 托管服务 | Cindy 云账户 | 使用 Cindy 的完整托管服务。参见[定价](https://cindy.app/#pricing)。 |
| 跳过登录 | 无需 Cindy 登录 | 在登录界面选择“跳过登录”以使用本地代理；应用随后将账户状态显示为“未登录”。在此状态下，基于服务器的功能不可用。 |

## 前提条件

- **Node.js** 22.x
- **pnpm** 10.x（尚不支持 v11）
- **Git LFS**

## 快速开始

贡献者设置、Git LFS、依赖更新和访问要求维护在 [`CONTRIBUTING.en.md`](CONTRIBUTING.en.md) 中。
插件通过 SkillHub 或手动安装。

最小入口点：

```bash
git clone https://github.com/makecindy/cindy.git
cd cindy
git lfs pull
pnpm install
```

## 开发入口点

```bash
# 中国大陆 Cindy 账户
pnpm restart:desktop:remote --region=cn

# 全球 Cindy 账户
pnpm restart:desktop:remote --region=global
```

远程开发使用你自己的 Cindy 云账户和现有登录状态，因此你可以在开发客户端的同时继续现有会话和工作。使用 `cn` 用于中国大陆账户，`global` 用于其他所有用户；不要依赖内部默认值。完整的桌面、移动、数据隔离和验证工作流程见 [`CONTRIBUTING.en.md`](CONTRIBUTING.en.md)。

登录界面上的“跳过登录”运行本地代理，无需 Cindy 账户（在应用中显示为“未登录”），并非连接到本地服务器。在此状态下，基于服务器的功能不可用。

**关于默认服务器：** 客户端默认连接到 Cindy 的官方云服务（端点清单见 [`config/endpoint.json`](config/endpoint.json) 和 [`config/endpoint.global.json`](config/endpoint.global.json)；桌面自动更新也来自官方 CDN）。这是有意为之 —— 外部开发者无需自托管服务器：在开发构建中使用你自己的 Cindy 账户登录，直接针对官方服务器进行开发/测试。

## 架构

- [`DESIGN.md`](DESIGN.md) — 视觉设计系统、颜色令牌和 UI 约定
- [`docs/README.md`](docs/README.md) — 完整文档和规则索引
- [`docs/auth-realm-routing.md`](docs/auth-realm-routing.md) — 组织 SSO 区域发现和会话端点路由
- [`CONTRIBUTING.en.md`](CONTRIBUTING.en.md) — 贡献者设置、验证和提交工作流程
- [`AGENTS.md`](AGENTS.md) — 工程规则、启动/运行时契约和模块边界
- [`docs/dev-rules/`](docs/dev-rules/) — 深入架构文档（例如 Orca 多代理编排）

## 贡献

贡献通过 Pull Request 合并到 `main` 分支。先阅读 [`CONTRIBUTING.en.md`](CONTRIBUTING.en.md)，然后使用 [`.github/PULL_REQUEST_TEMPLATE.md`](.github/PULL_REQUEST_TEMPLATE.md)。
每个提交都需要 [开发者原创证书](DCO) 签署（`git commit -s`）；每个 Pull Request 上的 DCO 检查会强制执行，且无需 CLA。
还请遵守 [`CODE_OF_CONDUCT.en.md`](CODE_OF_CONDUCT.en.md)。对于普通使用问题，请参阅 [`SUPPORT.en.md`](SUPPORT.en.md)；通过 [`SECURITY.en.md`](SECURITY.en.md) 私下报告安全问题。

## 安全

切勿将凭据或授权文件提交到工作树中。如果你发现安全问题，请按照 [`SECURITY.en.md`](SECURITY.en.md) 私下报告，而不是公开 Issue。

## 隐私与遥测

**官方分发构建** 包含 [TapDB](https://www.taptap.io/tapdb) 使用分析，用于产品级聚合统计（设备 / 操作系统 / 应用版本元数据；登录后与你的账户 ID 关联）。它**不**收集聊天内容、文件内容或工作目录数据。此外，在登录云账户时，客户端会向 Cindy 服务发送在线心跳（仅账户 ID、平台和版本）。崩溃转储保留在本地机器上，绝不会自动上传。

**从源代码构建？** 你不必保留分析功能：

- 移动端默认关闭 —— 没有在构建时注入 TapDB 凭据（`clientId` / `clientToken`），`apps/mobile/src/analytics/mobileTapdb.ts` 为空操作；
- 桌面端可通过移除 `apps/desktop/src/renderer/index.tsx` 中的 `initTapdb()` 调用完全剥离（实现位于 `apps/desktop/src/renderer/analytics/`）。

## License / 许可证

除非另有说明，本仓库中的源代码根据 [Apache License, Version 2.0](LICENSE) 许可。各个源文件不携带逐文件许可证头；仓库根目录的 `LICENSE` 文件具有管辖效力。

模型权重、数据集、提示词、商标和其他单独标识的材料可能受其自身许可条款约束，不自动包含在仓库级别的 Apache-2.0 授权范围内。第三方开源组件保留其自身的版权和许可。其归属声明和 SPDX SBOM 在 [`docs/legal/`](docs/legal/) 下管理，特定构建产物的输出位于 [`docs/legal/notices/`](docs/legal/notices/)。有关本项目的版权和归属信息，请参阅 [`NOTICE`](NOTICE)。