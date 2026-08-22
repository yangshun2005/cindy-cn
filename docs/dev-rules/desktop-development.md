# 桌面端开发、启动与验证

> **阅读时机**：在安装、启动、重启、调试或验证 `apps/desktop` 及其共享包时阅读。

本文档是桌面端开发命令及其使用条件的权威说明；可执行脚本以当前检出（checkout）根目录的 `package.json` 和 `apps/desktop/package.json` 为代码事实来源。

## Agent 启动入口

Agent 启动桌面端时，仅使用仓库根目录的安全包装命令，并显式选择目标区域。当用户仅说“启动开发版 / 启动测试版 / 查看当前更改”且未指定模式时，应使用当前工作树（worktree）的命名隔离沙箱，不要默认共享正式登录状态：

```bash
pnpm restart:desktop:remote --region=global -- --isolated=@worktree
pnpm restart:desktop:remote --region=cn -- --isolated=@worktree
```

`--isolated=@worktree` 会根据检出目录名派生一个稳定的沙箱名称（移除前导 `cindy-`，并附加路径短哈希，例如 `cindy-local-ollama-models` → `local-ollama-models-a1b2c3`）。同一工作树下次将使用此名称，登录状态将保留在此沙箱中。

仅当用户明确表示“共享登录 / 不要重新登录 / 使用现有数据 / 不要关闭当前实例”时，才添加 `--preserve-running`。不要将“用户未提及模式”理解为共享。

启动命令结束时，必须出现一行 `DESKTOP_DEV_VERDICT=ready` 才算成功；如果看到 `DESKTOP_DEV_VERDICT=failed` 或根本没有 verdict 行，则不得声称开发版已启动。失败时，将 `code` / `message` 提供给用户。如果存在 `next=` 且用户未明确要求必须共享，可以执行该 next 命令重试。不要给启动命令接上会吞掉退出码的管道。

桌面端连接的是您自己的 Cindy 云端账号（remote）。这与登录页面中免 Cindy 账号的“跳过登录”（应用内显示为“未登录”，无需账号即可使用本机 agent；代码内部标识仍为 `local` 模式）不是同一个概念。Agent 不得自行改用 `pnpm dev:desktop` 或 `pnpm dev:desktop:remote` 来绕过包装脚本。

启动包装器会先停止**当前检出**中已有的桌面端开发进程；其他工作树/命名沙箱的实例不受影响。必须尊重宿主提供的并行或保活工作流。脚本仅在宿主是**当前检出**的桌面端开发进程时拒绝重启（杀掉宿主会连这次启动一起终止）。宿主是正式版或另一个工作树时可以启动隔离沙箱；从另一个检出（checkout）的桌面端开发进程中启动共享实例仍会被拒绝，以避免两个进程争抢同一份正式配置文件。如果因当前检出宿主拒绝、或目标 userData 被其他检出占用而中止，不要更换命令绕过，应将 verdict 交给用户。

## 可选启动参数

两个 restart 命令都支持以下参数。脚本本身在不添加这些标志时，仍为共享库 + 正常调度（供用户在终端手动运行）。**Agent 例外**见上一节：用户仅说启动开发版时必须添加 `--isolated=@worktree`。这些参数仅对开发版生效，不影响用户机器上的正式版。

- `--region=cn|global`（默认 `global`）：切换构建身份与仓库内端点清单；中国大陆版必须显式传递 `--region=cn`，读取 `config/endpoint.json`。
- `--isolated` / `--isolated=<名称>` / `--isolated=@worktree`：使用独立的 userData 沙箱，数据库、登录状态、会话、定时任务和设备身份均与正式版完全隔离（首次需重新登录）；命名沙箱每个名称对应一个独立沙箱，名称限 `A-Za-z0-9_-`、≤32 个字符。`@worktree` 是保留名，根据当前检出目录派生沙箱名。当用户说“独立数据库／隔离数据／沙箱启动／不要动正式版数据”时使用；Agent 也将“启动开发版”归入此路径。**未合入主干的迁移（migration）必须在 `--isolated` 沙箱中运行，不得连接共享 userData**（参见 [`database-and-migrations.md`](database-and-migrations.md)）。沙箱（及任何开发版 userData 覆写）内不触发首次登录旧数据迁移（mToc）：不探测旧目录、不弹确认窗口、不将正式数据复制进沙箱。**`--isolated` 不得落在任一正式配置文件上**：若显式 `XDT_USER_DATA_DIR` 指向 CN / Global / Dev 任一正式目录（不限当前构建区域），启动器与主进程都将 fail closed。isolated 会更换独立的 deviceId，正式目录中的 refresh token 属于正式版设备，叠加在一起必然导致 `DEVICE_MISMATCH`，再删除磁盘数据会将正式版踢下线（2026-08-16）。
- `--passive`：定时任务被动模式，本实例不自动触发调度。当多开导致定时任务重复、需要让位给 primary 时使用。它可以与 `--isolated` 组合（隔离沙箱仅查看 UI、不运行定时任务是合法的）。**共享库只读契约不是这个标志本身**，而是解析后的正式配置文件 + passive 才落地。共享正式配置文件的 passive 实例对 userData 布局保持只读：不执行 owner-namespace 迁移（claim 推迟到下次独占启动），旧数据导入（`hasLegacyOwnerNamespaceClaim` 门控的 secret／IM／brain 数据迁移）一并等待。非 passive 实例在执行该迁移前也会先查询 `.dev-instances` 实例注册表（开发版与打包实例均会登记——开发版与正式版共享库双开受支持），若发现其他存活实例共享同一 userData，同样推迟——搬家式迁移必须独占 userData 才能执行，否则会打断仍在运行的旧版本实例（2026-07-23 slack-hook.json／网关凭证被搬走事故）。**auth 凭证同属此契约**：passive 共享实例不得**删除、作废或消费**整机共享的 auth 持久状态——磁盘 refresh token、服务端 device token（调用登出会连坐作废 primary 的那份）、relogin marker（一次性，被消费后 primary 将无法再看到）、canary flag、账号删除 receipt。其“退出登录”仅清除本进程内存状态（`authManager.ts` 中的 `isPassiveSharedUserDataInstance`）。代价是同机两个实例的登录状态可能不一致，这是有意的：passive 无权代表整机登出（2026-07-27 事故：MIGRATE_FAILED 的 passive 实例在 fatal 界面点击“返回登录”，删除了整机 refresh token，primary 在 19／46 分钟后的续期周期被强制重新登录）。
  约束的是破坏性动作，**并非写入本身**：passive 照常安排续期定时器，轮换后正常写回新的 refresh token——那写入的是有效凭证，primary 侧由 replacement-retry 消化。反之，让 passive 停止续期，会导致其 access token 过期后无替换途径（primary 的续期仅更新磁盘 token，不更新 passive 进程的内存状态，而直接走 `apiFetch` 的路径没有 401 refresh/retry）。
- `--preserve-running`：启动编排，而非运行期模式。默认 restart 本就不会关闭正式版和其他工作树，仅替换**当前检出**的旧开发版；本标志连这份旧开发版也保留，再并排开启一个共享库预览，并强制 `--passive`。启动前必须由 `.dev-instances` 存活记录证明已运行实例与目标区域一致，旧记录没有 region 或跨区域都会 fail closed。仅供能证明实例归属的上层编排，或用户明确表示“不要关闭当前实例／不要重新登录”时使用。仅支持 remote。禁止与 `--isolated` 或环境中的 `XDT_ISOLATED=1` 组合。共享实例若仅发现没有 realm 的旧版裸 refresh token，也不得猜测区域迁移或轮换，保持本进程登出，交由同区域独占实例完成凭证迁移。

若已手动设置 `XDT_USER_DATA_DIR`，应尊重用户值，不覆盖，也不探测或迁移正式区域目录。唯一例外：`--isolated` / `XDT_ISOLATED=1` 将该目录指向正式配置文件时直接拒绝启动。

正式版目录保持历史兼容：CN → `Cindy`，Global → `CindyGlobal`，不在启动时改名或迁移用户数据。非隔离开发版也使用当前区域对应的正式配置文件；`--isolated` 沙箱再按相同区域映射派生目录。**开发版写入器不得将正式配置文件升级到比安装版更新的 schema**：存在 pending migration 时拒绝启动，改用 `--isolated=<名称>`。`--preserve-running` / 共享库 passive 仍为只读。跨区域共享、登录状态迁移或旧版本回滚应使用显式隔离目录，避免不同构建误用同一配置文件。

### 并行多开开发版

restart 的 kill 作用域是**当前检出（worktree）**：仅停止本检出中的开发版进程，其他工作树/命名沙箱的实例一律保留（2026-07-30 约束：并行沙箱不得被另一个检出的启动器顶掉）。因此，并行多开的标准方式是：**每个实例一个独立工作树 + `--isolated=<名称>` 命名沙箱**，互不干扰地各自 restart。

配套护栏与工具：

- **userData 冲突门**：当目标 userData（按 `--isolated` 名称推导）已被其他检出的开发版实例占用时，restart 会在终止任何进程之前中止并列出占用进程——不代杀、不共享库。更换沙箱名称，或由用户自行停止该实例后重试。检测依赖辅助进程命令行上的 `--user-data-dir`，对方实例刚启动尚未启动辅助进程时可能漏检，属尽力而为。
- **CDP 端口**：开发版的 remote-debugging-port 固定为 9222，仅先启动的实例能绑定。后启动实例需要 CDP 调试界面时，使用 `XDT_CDP_PORT=<端口>` 覆写（仅数字生效，仅限开发版）。
- 同一检出内仍为单实例语义：restart 会替换本检出的上一个实例（不论沙箱名称），一个工作树同时仅运行一份开发版。
- 共享 userData 的并行（`--preserve-running` 被动预览）语义不变：不停止任何实例、强制 passive、禁止与 `--isolated` 组合，仅供能证明实例归属的上层编排使用。

Agent 自身仍仅走 restart 命令，不直接调用仅限人工使用的 `dev:desktop*`。共享同一 userData 多开时，非 primary 实例使用 `--passive` 让出定时任务调度（见上文）。

### 使用统计（TapDB）在开发版下不上报

开发版构建**默认不初始化 TapDB**，与用户是否同意《隐私政策》、统计开关是否打开无关。闸门在 main 侧 `analytics-settings-store.ts` 的 `isReportingBuild()`（`app.isPackaged !== true` 默认关闭），renderer 仅消费 `allowed` 这一结论。

原因：TapDB Web SDK 的设备身份（`device_id`）写在 renderer 的 localStorage 中，而 localStorage 按 **origin + userData 目录** 分家——开发版的 renderer 从 `http://localhost:<vite 端口>` 加载（并行多开时端口自增），`--isolated[=<名称>]` 与 `XDT_USER_DATA_DIR` 每条沙箱又各有一份。于是，一个开发者一天能凭空产生几十台“新增设备”，将线上新增设备／转化率／次日留存全部带偏（2026-07-26 复盘：某地区单人一天 78 台设备、新增账号 1、次日留存 2.6%）。开发版与发布版目前共用同一个 TapDB appId，只能在闸门上区分。

要验证上报链路本身时，手动设置 `XDT_TAPDB_DEV=1` 放行（严格等于 `1`，其他值一律视为关闭）。**这会将开发版数据打入线上应用，用完即撤，不要写入任何脚本或 `.env`。**

## 何时需要重启

- 修改 main、preload、MCP、原生依赖或包运行时代码后需要重启。
- 仅修改 renderer 时，优先使用现有实例的热更新，不重复重启。
- 不确定运行实例来自哪个检出时，先运行 `pnpm desktop:whoami -- --all` 核对。

## 分层验证

本节指导**开发过程中的增量验证**；提交（commit／PR）前的强制门禁以 `development-workflow.md` 的“提交前测试门禁”为准（仓库根 `pnpm test:unit:related` 与相关包的 typecheck 全部通过；CI 仍运行完整 `pnpm test:unit`）。开发过程中根据实际更改选择最小但充分的检查：

```bash
pnpm --filter desktop typecheck
pnpm --filter desktop lint
pnpm --filter desktop exec vitest run <测试文件路径>
pnpm --filter desktop test
pnpm build
pnpm test:unit:related
pnpm test:unit
```

- 修改 TypeScript 至少运行相关类型检查和定向测试。
- 跨模块、共享包、构建链或广泛重构时，再扩大到桌面端全量测试、构建或根级单元测试。
- 调整桌面端 Vitest worker 或测试分池前，先阅读 [`desktop-unit-test-performance.md`](desktop-unit-test-performance.md)，并使用其中的 benchmark 在相同测试范围下进行前后对比。
- 数据库迁移、协议、更新器、权限与用户数据另有高风险专项规则；命中时先阅读对应规则，不以本页命令替代专项验证。
- 记录实际执行和结果；未执行的高相关检查必须说明原因。