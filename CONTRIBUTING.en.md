<p align="right">
  <a href="CONTRIBUTING.md">简体中文</a> · <strong>English</strong>
</p>

# 贡献指南

感谢您为 Cindy 贡献代码、文档和反馈。本仓库是开源的 Cindy 客户端：包含桌面端和移动端应用及其共享包。服务器在单独的仓库中维护，不在本仓库的范围内。

## 开始之前

- 按照 [`docs/dev-rules/environment-setup.md`](docs/dev-rules/environment-setup.md) 了解支持的工具版本和安装步骤。
- 阅读 [`README.md`](README.md) 中的安装部分以及适用的[工程规则](AGENTS.md)。`AGENTS.md` 包含详细的工程约束；它不能替代本贡献指南。
- 参与社区时请遵循 [`CODE_OF_CONDUCT.en.md`](CODE_OF_CONDUCT.en.md)。对于普通使用问题，请参阅 [`SUPPORT.en.md`](SUPPORT.en.md)。
- 请勿提交凭据、令牌、授权文件、个人数据或生成的本地数据库。
- 公共仓库仅固定公共协议子模块。插件通过 SkillHub 或手动安装。请勿将凭据放入 Git 配置或仓库文件中。

## 获取代码并安装依赖

请按照[开发环境和依赖设置](docs/dev-rules/environment-setup.md)进行克隆、公共子模块、Git LFS 和依赖安装。该文档是安装命令的唯一权威来源；本指南不重复这些命令。

## 开发与验证

### 桌面端

参见[桌面端开发、启动和验证](docs/dev-rules/desktop-development.md)了解启动、区域选择、安全重启和验证命令。

### 移动端

参见[移动端开发、模拟器和验证](docs/dev-rules/mobile-development.md)了解模拟器、原生重建和验证命令。

### 验证

根据 [AGENTS.md](AGENTS.md) 中的风险分级原则选择检查项。使用各自开发规则中定义的桌面端和移动端命令。当更改涉及数据库、协议、端点、移动端权限范围或其他专业领域时，请阅读相应规则并运行其检查。每个 Pull Request 必须说明已运行的命令及其结果；并解释为何未运行任何高度相关的检查。

## 发起 Pull Request

1. 从最新的 `main` 分支创建短期分支，并保持每个 Pull Request 专注于一个明确的问题。
2. 使用 `<type>(<scope>): <short description>` 作为 Pull Request 标题，例如 `docs(readme): clarify local mode`。有关可用类型，请参阅[Pull Request 模板](.github/PULL_REQUEST_TEMPLATE.md)。
3. 审查完整的差异，确认其中不包含凭据、无关的生成文件或意外的子模块指针更改。
4. 填写[Pull Request 模板](.github/PULL_REQUEST_TEMPLATE.md)，包括范围、验证、风险和回滚信息。
5. 等待 CI 和审查；不要直接推送到 `main`。

欢迎通过 Pull Request 提交小型文档修复。对于架构、协议、数据库迁移、权限或用户数据的较大更改，请先开启 Issue 讨论范围和兼容性。

## 贡献许可（DCO）

本仓库根据 [Apache-2.0](LICENSE) 许可。根据许可证第 5 条，您有意提交以供包含的任何贡献均按 Apache-2.0 条款接受——无需单独的 CLA。

我们要求每个提交都带有[开发者原创证书](https://developercertificate.org/)签名（DCO 1.1；全文位于仓库根目录的 [`DCO`](DCO) 文件中）：使用 `git commit -s` 提交，这会附加一个 `Signed-off-by: Your Name <your@email>` 尾注，声明您有权根据这些条款提交贡献。尾注中的姓名和地址必须与提交的作者（或提交者）匹配——尾注是关于您本人的声明，不能代表他人做出。请勿提交您无权许可的代码（例如未经许可复制的专有代码）。

每个 Pull Request 上的 **DCO 检查**（[DCO GitHub App](https://github.com/apps/dco)）会验证 Pull Request 中的每个提交；合并提交和机器人提交豁免，且从不检查 DCO 要求之前的历史记录。使用 `pnpm check:dco` 在本地自行检查。如果某个提交缺少签名：

```bash
# 仅最新提交缺少签名
git commit --amend -s --no-edit

# 多个提交缺少签名（<base> 是 Pull Request 的基础提交）
git rebase --signoff <base>

# 之后更新 Pull Request
git push --force-with-lease
```

如果您不想重写历史——例如 Pull Request 已经包含值得保留的审查讨论——请推送**补救提交**。其消息必须逐字包含以下行，其中 sha 是被签名的提交的完整 40 字符 sha，且补救提交本身也必须带有您的签名：

```text
I, Your Name <your@email>, hereby add my Signed-off-by to this commit: <full 40-char sha>

Signed-off-by: Your Name <your@email>
```

两个提交的作者，以及该行上的姓名和地址，都必须完全匹配。要为他人提交签名：

```text
On behalf of Author Name <author@email>, I, Your Name <your@email>, hereby add my Signed-off-by to this commit: <full 40-char sha>

Signed-off-by: Your Name <your@email>
```

请注意，`pnpm check:dco` 故意比 Pull Request 上的门禁更保守：它只识别直接签名（不识别补救提交），且不豁免机器人提交，因为机器人状态取决于 GitHub 账户类型，无法离线确定。因此，本地通过意味着 Pull Request 通过，但反之则不然——使用补救提交或范围包含机器人提交时，以 Pull Request 上的 DCO 检查为准。

Git 没有自动签署提交的配置选项（`format.signOff` 仅影响 `git format-patch` / `git am`）。要一次性设置，请安装本仓库附带的 prepare-commit-msg 钩子。该钩子本身是 [`.githooks/prepare-commit-msg`](.githooks/prepare-commit-msg)——一个普通的 shell 脚本，安装前请阅读它。`pnpm dco:install-hook` 会将其复制到您的本地钩子目录；远程不会发生任何更改，删除已安装的副本即可卸载。您也可以使用 `git config core.hooksPath .githooks` 自行将 Git 指向该目录（这会接管整个钩子目录）。

## 安全问题

请勿在公共 Issue、Pull Request 或讨论中披露漏洞、凭据或可利用的细节。请遵循 [`SECURITY.en.md`](SECURITY.en.md) 中的私有报告流程。