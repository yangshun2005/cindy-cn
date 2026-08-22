# 插件运行时安全与作者契约

> **状态**：权威开发规则（authoritative）
> **读取时机**：新增或修改插件（`.cindy`）的运行时、沙箱、权限、能力 slot、面板供片、
> 网络／凭证／文件交接，插件作者可见的身份卡、管子协议、打包与编写手册，或批准状态、
> 安装布局、指纹格式等**存量安装读得到的任何东西**之前

本文治理 Cindy 中运行的插件——以 `ghost.json` 为身份卡的 `.cindy` 包。它约束三件
互为支撑的事：**运行时权限安全**（宿主如何隔离并授权插件）、**存量插件兼容**（升级
不得让用户已装的插件失效或需要重装）和**作者契约同步**（改了作者可见的能力，必须同步
编写手册与校验）。Electron 进程与协议安全另见
[`electron-security-and-process-boundaries.md`](electron-security-and-process-boundaries.md)，
媒体字节交接另见 [`media-storage-and-protocols.md`](media-storage-and-protocols.md)，插件在
产品中的定位见 [`../product-rules/core-product-principles.md`](../product-rules/core-product-principles.md)。

> **术语**：产品与对外措辞统一为「插件」，`.cindy` 即插件包；不再使用旧的概念称呼。
> 代码标识仍沿用历史 `Ghost` / `cindy-brain` 命名（目录 `cindy-brain/`、`GhostRuntime`、
> `ghost.json`、`cindy-ghost://`、`ghost_forge_guide` 等），与产品术语「插件」指同一事物；
> 引用代码时照实使用这些标识，不因措辞改写。

> **增量适用原则**：运行时沙箱与权限的安全不变量（下节 2–4）与存量插件兼容红线
> （下节 5）对**所有**触及相关路径的改动都生效，不因是存量代码而豁免。作者契约同步
> （下节 6）在改动命中作者可见契约时触发；不要求为统一形式专项重构无关存量。

## 事实来源

| 内容                                                  | 权威来源                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| ----------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 编写手册（作者唯一教材，现拿现读）                    | `apps/desktop/src/main/cindy-brain/forge.ts` 的 `FORGE_GUIDE`，经 `ghost_forge_guide` 工具下发                                                                                                                                                                                                                                                                                                                                                                                           |
| 身份卡字段与校验、管子协议类型                        | `apps/desktop/src/shared/ghost.ts`（`validateGhostManifest`、`cindy.send` / `cindy.onHostMessage` 类型）                                                                                                                                                                                                                                                                                                                                                                                 |
| 打包限制                                              | `apps/desktop/src/main/cindy-brain/forge.ts` 的 `packGhostDir`                                                                                                                                                                                                                                                                                                                                                                                                                           |
| 运行时、沙箱进程与生命周期                            | `apps/desktop/src/main/cindy-brain/runtime/GhostRuntime.ts`、`GhostManager.ts`                                                                                                                                                                                                                                                                                                                                                                                                           |
| 安装批准事实(receipt / 技能快照 / revision)           | `apps/desktop/src/main/cindy-brain/ghostInstallReceipt.ts`，批准态投影见 `shared/ghost.ts` 的 `GhostInstallApproval`                                                                                                                                                                                                                                                                                                                                                                      |
| 能力 slot（网络／通知／确认／文件系统／技能／宿主等） | `apps/desktop/src/main/cindy-brain/networkSlot.ts`、`notifySlot.ts`、`badgeSlot.ts`（未读角标，落盘账本 `ghostUnreadStore.ts`）、`confirmSlot.ts`（往返桥 `ghostConfirmDialogBridge.ts`，renderer 落地 `cindy-brain/GhostConfirmDialogHost.tsx`）、`fsSlot.ts`、`cindySlot.ts`、`skillSlot.ts`、`agentSlot.ts`、`errandSlot.ts`（派活执行链在 `maker-ipc/ghostErrandRunner.ts`，每插件配置在 `errandPrefsStore.ts`）、`iosSimulatorSlot.ts`（当前台前任务的公开状态与 Host viewer 入口）；library 槽（持久作品库：文件层/SQLite 语句门/binding/迁移/回收站）见 [`plugin-library-storage.md`](plugin-library-storage.md)，主实现在 `libraryVault.ts`、`librarySlot.ts`、`libraryDbCore.ts` |
| 面板供片、注入主题 token 与协议                       | `apps/desktop/src/renderer/cindy-brain/ghostPanelTheme.ts`、`cindy-ghost://` 分支                                                                                                                                                                                                                                                                                                                                                                                                        |
| 权限注入／更新确认 UI                                 | `apps/desktop/src/renderer/cindy-brain/GhostPermissionList.tsx`                                                                                                                                                                                                                                                                                                                                                                                                                          |
| 远程／手机版能力准入白名单                            | `packages/device-link/src/allowlist.ts`                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| 行为与安全不变量                                      | `apps/desktop/src/main/cindy-brain/__tests__/`、`forge.test.ts`                                                                                                                                                                                                                                                                                                                                                                                                                          |

文档与实现冲突时以代码为准，但必须在同一改动内同步修正本文与手册。

## 1. 插件形态与代码术语

- `.cindy` 是以 `ghost.json` 为身份卡的插件包，现行唯一形态为 `kind: 'chip'`。
- 代码目录与运行时使用 `cindy-brain` / `Ghost` 命名，**不得重新引入已退役的 cartridge
  声明型兼容层**。
- `cindy-` id 前缀保留给随包官方插件，第三方插件不得占用。

## 2. 运行时沙箱与进程隔离

- 每个运行中的插件使用独立 Electron 沙箱进程与专属 session partition。沙箱禁止直接访问
  Node、宿主文件系统和网络。
- 插件只允许读取自身安装目录内、经安全相对路径校验的静态资源，不得越权读取其它目录。
- 逻辑页只能经最小 `contextBridge` 管子申请主机能力；面板 webview 保持零特权桥。
- 主机按 `webContents` 绑定反查真实 ghostId，**不信任 sender 自报身份**。
- 沉睡、抽离和主机退出必须终止对应沙箱；沙箱崩溃只由 `GhostRuntime` 收敛，不得带崩
  主应用。

## 3. 权限即授权边界

- 所有能力必须先在 manifest 声明 slot，通过同一套校验，并在注入／更新确认框中逐项
  如实展示，再由 host 代码强制授权。**prompt 不构成安全边界**，前端展示与确认框文案
  也不构成授权。
- 新增或修改 slot 时，除同步编写手册与校验（下节 6）外，还必须同步 shared 类型、
  preload／host handler、权限 UI（`GhostPermissionList.tsx`）、错误边界和测试。
- **授权事实由 Host receipt 持有，不由安装目录持有。** 一次明确的安装／更新确认写出
  一份 receipt（`ghostInstallReceipt.ts`），落在**安装根之外**的 owner-scoped 状态根里，
  钉住这次批准过的 manifest、trust、启停态和一个随机 `revision`；`GhostManager.list()`
  只从 receipt 取这些字段，安装目录里的 `ghost.json` / `.cindy-trust.json` / `.disabled`
  退化为旧版本兼容镜像。**唯一的非对称例外是启停态**：`.disabled` 镜像在读取时只往
  停用方向合并（`enabled = receipt.enabled && !镜像存在`）——停用必须永远能成功，状态根
  不可写时镜像是 `setEnabled(false)` 唯一还能落笔的地方，只读 receipt 会让那次停用在
  重启后静默复活；镜像**不能**把插件往启用方向翻，重新启用只有 `setEnabled(true)` 成功
  写 receipt 一条路。理由是可变安装目录曾经就是授权事实本身：就地改写
  `ghost.json` 能让权限 diff 显示"无新增"，未确认的 slot 因此拿到运行授权。
  - 没有 receipt（旧安装）或 receipt 损坏 = **不构成运行授权**：一律按停用列出、
    不许启用、不参与技能落链。但"不构成运行授权"只是**当下状态**，不等于"该由用户
    重装"：这两种情况都必须先走第 5 节的存量迁移（能从旧版授权事实重建出等价 receipt
    就自动重建，用户无感），**只有迁移读不出或自相矛盾时**才落到一次完整重新确认
    （`diffInstalledGhostPermissionItems` 在无批准基线时把候选包的**全部**权限当新增项
    展示）。UI 必须如实说出这个状态并给出恢复入口，不能让它看起来只是"被用户关掉了"，
    也不能只留"去市场重装"这一条出路。
  - 通往"不构成运行授权"的还有第三条路：**撤销陈旧批准失败时的进程内隔离**。撤销的
    契约是"返回后该插件一定不再被授权运行"，所以删不掉 receipt（状态根不可写——与写
    批准失败同一个成因，指望再往状态根写点什么表达"已失效"并不可靠）时退回内存标记，
    由读批准状态的**唯一入口**统一投影成 `invalid`。读批准状态的所有消费方都必须走
    那个入口：各自直接读 receipt 会让隔离在某条路径上失效。撤销同时要熄灯运行中的
    实例（runtime / node broker / agent slot 三连），否则"不再被授权运行"对已经跑起来
    的进程不成立。下一轮启动对账成功即自愈。
  - 跨进程更新事务用 `ghostInstallApprovalToken()` 把批准态投影成 token：Renderer 把
    确认时看到的 token 回传，Main 重新读状态比对，不一致就拒（`state-changed`）。
    token 是前置条件不是凭证——真值一律由 Main 现读。
  - receipt 保证的是**授权事实**，**不是安装内容此后一直没被改过**：逻辑页代码仍从可变
    安装目录加载，`packageSha256` 只是批准时点的来源指纹、运行时不校验（见第 7 节）。
- **Forge 的源码区与 Host 受管根互斥。** `ghost_forge_scaffold` / `ghost_forge_pack` 的目标
  必须是当前会话工作目录里的独立作者目录；命中安装根或批准状态根一律拒
  （pack 返回 `SOURCE_IS_INSTALLED_PLUGIN`）。判定按 realpath 比对受管根，同时挡住大小写
  折叠与软链／junction 别名。理由不是洁癖：在已安装目录里就地制作"更新包"会让版本与
  权限 diff 以被改过的现场为基线，把未经确认的 manifest 送进运行时授权。
- `skill` 槽是唯一**越出沙箱**的能力：技能指令由主 Agent 以用户全部权限执行、全局
  生效、不随 workdir 级停用隐藏。其安全边界是**声明一致性**（manifest 里的
  name／description 必须与 SKILL.md frontmatter 逐字一致，`skillSlot.ts` 的
  `checkSkillMdConsistency` 是唯一裁判，打包与装入、以及批准快照三侧共用；注意它**只**
  校验 frontmatter 的 name／description，正文与辅助文件不在它的判据里）+
  **批准快照与字节指纹**（确认时把技能目录逐字节拷进
  `<状态根>/skill-snapshots/<id>/<revision>`，只收普通文件，同时把逐 item 的内容
  指纹钉进 receipt 的 `skillContentSha256`——装入/更新时该指纹取自 **`.cindy` 包的
  内存投影**(inspect 时已被 `packageSha256` 钉住的那份字节),不从已发布的可变安装
  目录首读:publish 与首次 hash 之间被换的字节应当在快照对账时被拒,而不是被首读钉成
  批准基线;确认框看到的 SKILL.md 必须就是 Agent 之后
  读到的那份，所以共享技能根的链接指快照而不是可被改写的 `cindy-brain/<id>/<dir>`。
  快照缺失需要从安装目录重建时，**顺序本身就是安全性质**：先把字节复制进状态根的
  临时目录，再对**临时目录里那份即将成为快照的字节**做全部权威校验（尺寸上限 →
  指纹逐字节比对 → frontmatter 一致性），通过才 rename 就位。**不得改成"先校验安装
  目录、再复制"**——安装目录随时可被同权限进程改写，校验与复制各读一次就有一个可换
  字节的窗口，复制出来的快照可能不是被校验过的那一份。同理，复制前对安装目录做的
  任何预检只是"早失败"优化，**不是安全边界**；尺寸上限必须排在算指纹之前——上限要在
  权威路径上真正生效，且不为一份注定被拒的字节先付一整趟读取成本。指纹计算一律流式
  喂入、不整份读进内存（技能目录里除 SKILL.md 之外的文件没有尺寸上限，整份读会被一个
  塞进来的超大辅助文件撑爆）。只靠 `checkSkillMdConsistency` 拦不住"frontmatter 不动、改写正文或塞
  辅助文件"，那会把一份没人确认过的指令在一次启用里固化成已批准快照并全局挂链。
  对不上一律拒、退回完整重新确认，不许就地自愈成新批准；`skillContentSha256` 因此是
  **运行期判据**，与只作审计用的 `packageSha256` 不同，且必填——留"字段缺失就跳过
  校验"的可选口子等于给漂移开一条绕过路径）+ **链接对账**（`reconcileGhostSkillLinks` 只增删"目标落在
  安装根或批准状态根内的 symlink／junction"，绝不触碰真实目录与外来链接；
  启用挂链、停用／卸载撤链、断链自愈）。快照目标带 revision，因此每次更新都换目标、
  靠对账重指，旧 revision 在 receipt 提交后回收。改动技能落链、命名
  （`<id>--<name>`，name 侧禁 `--`）、快照或对账判据前，必须先读 `skillSlot.ts`
  头注释并保持上述不变量；`approvalStateRoot` 是必填项——漏给会让指向快照的活链接被
  判成外来链接而永不撤链，停用／卸载后技能仍对主 Agent 生效。
- 收敛方向不对称：**启用需要有效批准状态，停用必须永远能成功**。停用是安全方向，不
  能因为快照缺失之类的环境问题把插件卡在"既不能用也不能关"。
- **「插件内容目录怎么读」只有一份判据：`ghostContentTree.ts`。** 条目类型判定
  （`classifyGhostDirEntry`，一律 `lstat`、链接与非普通条目显式归类，**不信 Dirent 的
  类型位**）、清单相对路径的逐段解析（`resolveGhostContentPath`，中间段是链接一律拒）、
  内容树收集与指纹格式（`collectGhostContentFiles` / `hashGhostContentFiles`）都在这里，
  各调用方之间的差异只允许以**显式策略参数**表达：点开头条目算不算内容
  （技能目录 `include` ／安装目录与种子 `skip`）、非普通条目是 `throw`（授权判据）还是
  `flag`（对账判据——收敛动作是重新播种而不是拒绝）。
  - 理由是实测的复发史：同一条判据曾经在技能指纹、快照拷贝、安装目录漂移指纹、随包
    种子指纹、种子复制、Forge 打包收集六处各写一遍，还有五处各自 `path.join` 后再判
    一次类型，分别用 Dirent 类型位／`lstat`／`stat`／realpath 钳制实现。于是每轮审查都
    能在其中一处找到没覆盖的角落，补一处、下一轮换另一处。
  - 新增任何"读插件内容目录"的代码一律从这里取判据，**不要就地 `readdir` + `isDirectory()`
    或 `stat` 直读**。只 `lstat` 最终段等于没判：中间段被换成软链／junction 时 OS 会
     静默穿透，最终段报的是"真目录、非链接"，字节却来自插件目录之外。
  - `hashGhostContentFiles` 的摘要编码必须保持无歧义 framing（当前为
    `cindy-ghost-content-v2` + UTF-8 路径长度前缀 + 每文件摘要），不得恢复成
    `path + NUL + bytes + NUL`；文件内容本身允许包含 NUL，分隔符编码会产生不同文件树
    的等摘要。该编码升级同步 bump `GhostInstallReceipt` schema；旧 receipt 必须
    fail closed 并重新确认，不能拿旧摘要继续授权。**编码 bump 属于第 5 节的存量兼容
    场景**：先按**旧编码**核对旧摘要与当前安装字节，对得上就原地重算成新编码并升级
    receipt（安全水位与旧版本已提供的保证等价），只有对不上（真漂移）才 fail closed 到
    重新确认——不得把一次纯格式升级直接变成全体用户重新确认。
  - 同理，"源目录与受管根的包含关系"必须**双向**判（既不能落在受管根内，也不能是受管
     根的祖先）：单向判定下只要在 owner 数据目录里放一个 `ghost.json`，递归打包就会把
     已安装插件字节、批准 receipt 与技能快照打进 `.cindy`。
  - 随包种子是第一方输入；发现链接、junction、FIFO 等非普通条目必须整颗跳过并告警，
    不得在复制时静默丢弃后继续写批准 receipt。

## 4. 网络、凭证与资源交接

- network 只允许 manifest 白名单域名；凭证由主机保险库