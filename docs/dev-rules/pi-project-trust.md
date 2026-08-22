# Cindy 管理的 Pi 项目信任契约（#2013）

本文件定义了 PR4（#2014）可消费的输入/输出契约；本 Issue 不改变 Pi 运行时行为。

## 真源与生命周期

- 信任输入只能来自 Cindy 已有、可审计的项目批准。当前 `origin/main` 尚未提供通用的项目批准存储，因此实现必须通过宿主注入 `PiProjectApprovalSnapshot`，不得将 `permissionMode`、工具审批、MCP 批准、插件启用状态或 Pi 用户设置解释为项目信任。
- 批准在新建、重启、Fork 或切换到新 `workingDir` 时重新求值；运行中的 Pi 进程使用启动时的快照。撤销或失效对下一次新会话生效，不声称热卸载已加载的资源。
- `workingDir` 与 git 仓库根目录必须先进行 `realpath`/规范化，宿主必须同时提供可信的 `platform`（`posix` / `win32`），不得从路径字符串猜测或默认使用 POSIX。POSIX 规范字节只有在能无损往返 UTF-8 时才可标记为 `utf8-lossless`；Windows 规范路径只有在能无损往返宿主 Unicode 字符串时才可标记为 `utf16-lossless`。编码标记与平台不匹配、包含替换字符或无法证明无损时必须标记为 `unavailable`，不得使用有损字符串生成批准密钥。解析失败、目录消失、仓库边界变化、符号链接指向变化均采取失败关闭。默认作用域是 `repo-root + workingDir`；只有批准明确声明 `repo-root` 才能让同一仓库的多个 workingDir 共享批准。`extraDirs`、引用目录和其他工作区根目录不继承。
- 当前纯函数仅在宿主明确提供 `windowsCaseComparison: ordinal-insensitive` 时，对 ASCII Windows 规范路径的比较键进行分隔符、扩展长度前缀和大小写归一化；若宿主报告 `case-sensitive`，则保留比较键大小写；缺省、`unavailable` 或无法证明比较语义时统一失败关闭。比较键归一化不得覆盖宿主提供的规范 I/O 路径，符合条件的输出必须保留扩展长度前缀及尾随空格/点。`ordinal-insensitive` 下的非 ASCII Windows 路径因 JavaScript Unicode 折叠不等同于 Win32 序数比较，也必须失败关闭为 `unavailable`，直到宿主提供独立的 Win32 比较标识。
- `projectKey` 为 `${canonicalRepoRoot}\0${canonicalWorkingDir}`；Windows 比较标识决定是否折叠大小写，分隔符始终归一化。批准的 `scopeKey` 对 `repo-root` 是规范根目录，对 `working-dir` 是同样的复合键。

## 状态与资源边界

纯函数输出的状态只有 `approved`、`unapproved`、`revoked`、`stale`、`unavailable`。文件存在或扫描器命中只能形成资源 `discovered`，不能宣称 Pi `loaded`；`loaded` 只能由 #2011 的 `get_commands` 运行时清单证明。

批准决策只产生 PR4 的候选输入：

- `skills`：只有调用方显式证明 `explicitSkills: true`，且在新增的 `canonicalSkillEvidence` 中为每条发现路径提供精确的 `discoveredPath`/`canonicalPath` 配对的 1:1 realpath 证据，并验证规范路径位于批准的仓库根目录内时，才可将包含 `SKILL.md` 的技能目录交给 PR4 装配；单文件 Markdown 不属于当前 Cindy 项目发现契约，保持 `discovered`。旧的 `canonicalSkills` 字符串数组仅为兼容保留，不能单独构成资格证据。宿主必须先从 `discovered.skills` 中排除用户、`extraDirs`、引用目录和其他工作区根目录，规范证据不能替代该来源过滤。缺少证据、配对不一致、仓库外符号链接目标或非规范路径时保持 `discovered`。符合条件的输出保留规范 I/O 路径，不使用比较键替代；PR4 必须将符合条件的目录完整物化到每会话隔离的 `configHome`，只将快照路径交给 Pi，任一复制或复核失败则整组失败关闭。能力位只有严格的布尔 `true` 才生效，缺省或其他真值均为 false。它不代表 Pi 已经 `loaded` 这些资源；
- `settings`：只有经字段白名单投影（输出 `settingsProjection.values`）、为每条发现路径提供带精确 `discoveredPath`/`canonicalPath` 配对的 1:1 realpath 证据并验证规范路径位于批准的仓库根目录内，且同时证明 packages/extensions 关闭时才可装配；`settingsProjection.sourcePath` 必须是 `discovered.settings` 中的原始词法路径，成功后输出才改写为对应的规范 I/O 路径。缺少证据、配对不一致、仓库外符号链接目标或非规范路径时保持 `discovered`。本契约默认不允许原始 `.pi/settings.json`。当前白名单只允许固定的 Pi v0.83.0 的 `compaction.reserveTokens` / `compaction.keepRecentTokens` 非负安全整数；未知字段、未来新增字段、`defaultProjectTrust`、provider/auth、路径及资源加载配置均失败关闭。输出是深克隆并冻结的启动快照；`eligibleSettingsPaths` 仅为兼容字段，包含投影使用的规范 `sourcePath`，不能将发现到的原始 settings 路径当作可装配输入；空投影保持 `discovered`。
- `packages` / `extensions`：始终 `discovered` 或 `blocked`，不得安装、加载或执行。

## 启动与隔离不变量

- 不传 `--approve`，不写用户 `~/.pi/agent/trust.json`，不设置 `defaultProjectTrust=always`；trust.json、`defaultProjectTrust` 或等效配置的装配属于 #2014。
- 不写或复用整个用户 `~/.pi/agent`，不继承 auth/provider/settings/trust/凭证。每会话 `PI_CODING_AGENT_DIR` 隔离、临时目录和并发会话输入隔离保持不变。
- `launch` 输出固定为 `approve:false`、`writeTrustJson:false`、`inheritUserPiHome:false`、`allowPackages:false`、`allowExtensions:false`。纯函数不读配置、不写文件、不启动 Pi。
- 如果 Pi 无法同时做到“信任 skills/settings 但阻止 packages/extensions”，#2014 必须缩小为显式仅 skills，或阻断项目 settings；不得用 `--approve` 绕过。

## 与既有边界对照

- #1729：只允许 Cindy 已批准项目映射；禁止无条件 `--approve`、复用整个用户 Pi 目录与默认开放 packages/extensions。
- #1705：packages/extensions 在非 TUI 宿主下的执行与呈现契约仍未解决；本契约不重开该范围。
- #1967：隔离 `PI_CODING_AGENT_DIR` 是有意的并发/凭证边界；本契约不通过复制或链接用户 Pi home 规避隔离。
- #2030：以 Pi v0.83.0 `get_commands` 夹具作为资源发现事实基线；#2053/#2011 的清单类型合入后，只需复核字段接线，不复制其未合入类型。