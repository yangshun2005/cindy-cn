# 客户端日志上报：采集、脱敏与崩溃补传

> **状态**：权威开发规则（authoritative）
> **读取时机**：修改客户端日志采集／脱敏／上报链路，修改 `main/logger.ts` 的 main 日志行
> 格式，或修改崩溃判定与待补传标记之前

本文管**日志怎么离开用户的机器**。日志模块本身的约定（统一 logger、不裸 `console.log`）见
[`engineering-conventions.md`](engineering-conventions.md) §1；凭证不落盘见
[`credentials-and-local-storage.md`](credentials-and-local-storage.md)；开关的默认值与
override 语义见 [`configuration-and-overrides.md`](configuration-and-overrides.md)。

需求与设计正文：[`../client-log-upload-requirements.md`](../client-log-upload-requirements.md)、
[`../client-log-upload-implementation-plan.md`](../client-log-upload-implementation-plan.md)。

## 事实来源

| 内容 | 权威来源 |
|---|---|
| 记录边界格式（写侧＋读侧共用） | `apps/desktop/src/shared/mainLogRecordFormat.ts` |
| 本地日志保留天数 | `apps/desktop/src/shared/logRetention.ts` |
| 跨进程契约与上传编号形态 | `apps/desktop/src/shared/logUpload.ts` |
| 来源白名单 | `apps/desktop/src/main/log-upload/sourceAllowlist.ts` |
| 脱敏规则 | `apps/desktop/src/main/log-upload/redact.ts` |
| 体量上限 | `apps/desktop/src/main/log-upload/limits.ts` |
| 上报目标（构建期注入的解析侧） | `apps/desktop/src/main/log-upload/logUploadTarget.ts` |
| 上报目标（构建期注入的校验侧） | `scripts/shared/log-upload-build-env.mjs`、配置骨架 `config/log-upload.json.example` |
| 待补传标记 | `apps/desktop/src/main/log-upload/pendingMarkers.ts` |
| 崩溃判定 | `apps/desktop/src/main/lifecycle.ts`（`isFatalShutdownReason` / `onFatalShutdown`） |

## 1. 三条不变量

这套东西的正确性依赖三条不变量。破坏任一条的后果都不是「功能不好用」，而是隐私事故或崩溃
现场静默丢失。

### 1.1 记录边界是安全不变量，不是排版

`main-<date>.log` 是纯文本按行解析的。上报侧按行首特征
（`MAIN_LOG_RECORD_HEAD_RE`）识别一条记录的起点，并据此判断该记录的来源 scope 是否在放行
名单内。

因此**写侧必须保证：除记录首行外，没有任何行以边界特征开头**。做法是 `emit()` 对 `msg` 调
`escapeMainLogContinuationLines()`（每个续行前置一个空格）。

不做这件事的后果：一条被封禁来源（例如把用户输入写进 debug 日志的功能模块）的多行内容里，
可以嵌入一个伪造的「放行来源」记录头，把对话正文伪装成基础设施日志送出去。

**这两侧是同一条不变量的两半，必须同时满足。** 改动 `logger.ts` 的行格式、改动读侧的切分
逻辑、或者「优化」掉那个前置空格，都必须同时确认另一侧。

存量文件的过渡由**格式哨兵**闭合：logger 每次打开 main 当天文件后写一行
`#cindy-log-format:2`。读侧的信任判据是「**文件第 0 字节就是哨兵**」——满足则整份文件按转义
格式解析，不满足则整份文件一条都不采（`startsWithFormatSentinel`）。

⚠️ **判据必须是「第 0 字节」，不能是「文件里出现过哨兵」。** 哨兵行的形状（记录头 +
`[logger]` + 固定串）完全可以由日志正文逐字构造：只要判据是「出现过」，未转义的存量文件里
就能嵌一行伪造哨兵、后面跟若干伪造的放行 scope 记录头，把对话正文当基础设施日志送出去
（2026-08-04 review 连续两轮命中这一点：先是 `indexOf` 子串匹配，改成整行精确校验后仍可
伪造）。而第 0 字节不可能是正文——新版本新建当天文件后的第一次写入就是哨兵，旧版本的第 0
字节永远是它自己那条真实记录。

代价是**跨越升级那一刻的当天文件整份不可上报**（前半段旧版本写的、后半段追加在哨兵之后）。
只影响每台机器升级当天的那一个文件：更早的纯存量文件本来就一条都不放行，之后的文件哨兵都在
第 0 字节。这笔账是有意认的——用一天的可观测性换一条能自证的信任链。跳过时计入
`stats.filesSkippedLegacyFormat`，便于把「升级当天采到 0 条」一眼归因。

不要为了「多采一点历史」把判据放宽成「出现过哨兵」。

### 1.2 白名单方向不可反转（deny-by-default）

来源名单是**白名单**：只放行确定与用户内容无关的基础设施来源，未知来源一律丢弃。

不能改成黑名单。功能模块会在 debug 级别把用户输入写进日志——语音听写草稿、命令行、搜索
关键词、界面上最后一条用户消息……黑名单逐个封禁不收敛，永远有下一个。

推论：

- **新增诊断来源需要显式加入名单**，这是有意为之的代价，不是待优化项。
- 名单**只增不减**方向上的「增」也要过 review：每条都要写明理由，理由写不出来的不该加。
- `console` 永远不进名单：它是第三方库与任何漏网 `console.log` 的兜底落点，内容不可控。
- **一个 root 只要有任何子 scope 会打用户内容，这个 root 就不该做根放行**，改用
  `ALLOWED_EXACT_SCOPES` 精确匹配。根放行 + 逐条排除是个陷阱：**新增的子 scope 默认是放行
  的**，方向与 deny-by-default 相反。设备互联就是这么漏的——`device-link` 曾作为根放行，
  排除表挡住了 media / mirror 那几个，却漏了 `device-link:ipc`（它在镜像缓存清理失败时把本地
  缓存文件路径写进日志）。`DENIED_SUB_SCOPES` 现在只作为纵深防御保留。
- 需要某个 scope 里的一部分诊断信息、而它整体又混着用户内容时，**拆 scope**，不要整体放行。
  已有先例：`renderer-console` 从 `renderer-guard` 拆出（前者是渲染进程任意 console 正文，
  后者只有加载失败信号）；device-link 的连接层与 IPC 层分开对待；`auth-adapters` 拆出**两个**
  denied 子 scope——`auth-adapters:asset-prep`（全局 skill/plugin/marketplace 资产准备的告警，带
  用户自选 skill·marketplace 名与绝对路径 `cannot link skill X from <path> to <path>`）与
  `auth-adapters:cred-path`（凭证文件 icacls/chmod/rm/硬链失败诊断，带 `auth.json` 等绝对路径——
  脱敏只抹用户名段、路径其余部分仍在），根 `auth-adapters` 只剩不带用户身份/路径的凭证生命周期
  诊断（2026-08-06 review）。**拆 scope 只改上报可见性，本机日志照常写全**。
- 渲染进程转发的日志整类不放行——`writeFromRenderer()` 强制 `r:` 前缀，而匹配是根锚定的。
  不要把匹配改成裸 `startsWith`。

### 1.3 待补传标记：代次令牌 + 原子清除

开发版与正式版共享同一份 userData，两个实例可能并发读写同一批标记。因此：

- 每条标记带**唯一代次令牌**，且令牌进文件名。仅靠时间戳在同毫秒的并发写下会误判成同一条。
- 认领靠**同目录 rename**（原子替换），抢输的实例拿到 ENOENT 直接跳过。
- 清除只删**自己那个 claim 文件名**，绝不会把另一个实例刚写的新崩溃标记误删。
- **仅在「确实传成功且非空」时才清除**。上传失败、采到 0 条、授权读不出来，一律还原标记。
- 授权被关闭时**清空全部**标记（含 claim 文件）——用户关掉授权后不得在下次启动补传。

## 2. 四层收窄（改任一层前先读这一节）

上报内容的边界靠四层，**不是单层过滤**。日志由客户端直接送到日志服务，中间没有服务端环节可
以再过一遍——客户端这套就是唯一防线。

| 层 | 决定什么 | 实现 |
|---|---|---|
| 1 源白名单 | 读哪些文件 | `collect.ts`：只构造 `main-<date>.log` 与 logs 根的 `agent-<date>.ndjson`。`sessions/**` 与 `*cc-debug.raw.log` **永不构造路径** |
| 2 记录白名单 | 放行哪些记录 | `sourceAllowlist.ts`（deny-by-default，见 §1.2） |
| 3 正则红线 | 抹除敏感片段 | `redact.ts`（宁可多抹，不可漏） |
| 4 字段白名单 + 截断 | 带出哪些字段 | 只有 `ts` / `level` / `src` / `scope` / `msg` 五个字段离开本机 |

`agent-<date>.ndjson` 只在崩溃路径附带，且**只取 `source === 'proxy'` 且 scope 落在 proxy 根下
的记录**（双闸）。同一文件里还有 `maker` 源的日志，那些可能带 agent 提示词与用户内容。

⚠️ **proxy 记录不能原样搬 `msg`，必须逐字段重建**（2026-08-04 review P1）。proxy 自己会把
请求体与上游错误体写进日志上下文：

- `logger.debug('▶ inbound request from client', { …, body: dumpBody(rawBody) })` ——
  `XDT_PROXY_DUMP_REQUEST_BODY=1` 时带**完整请求体**，对 anthropic-compat proxy 就是整个
  prompt（对话正文 + 被读进上下文的文件内容）；
- `logger.warn('◀ upstream response (non-2xx)', { …, body: dumpBody(errBody) })` ——
  只要 debug 等级开着就带上游错误体，而它发在 **warn** 级，**光按等级过滤挡不住**。

而 `logger.emit()` 是 `util.format(...args)`，上下文对象会被渲染进 `msg`。所以三道一起上，
缺一不可（`agentLogReader.ts`）：

1. **等级闸**：只放行 info 及以上；等级缺失或不认识按 debug 处理（未知不该比明确的 debug 宽松）；
2. **标记截断**：只取 `msg` 里渲染对象之前那截字面量（对象里的值因此不可能进标记），截断；
3. **标量字段白名单**：`PROXY_FIELDS` 逐个键配窄正则取值，`body` 这类名单外的键**没有出口**，
   与等级无关。形状写窄是要点——用 `.*` 取值等于把「这个值安全」的判断让给写日志的人。

新增可带出的 proxy 字段等同于放宽隐私边界：要论证该键在 debug 打开时也不可能承载用户内容。

脱敏规则**只增不减**：放宽任何一条（缩小匹配范围、提高最小长度、去掉某个形态）都视为隐私
变更，需要重新评审。按**形态**写规则而不是按厂商——厂商清单永远滞后于新出现的 key 形态。

规则**顺序本身是约束**，不是排版：`redact.ts` 的注释逐条写明了理由，改顺序前先读。已经踩过
一次的坑（2026-08-04 review P1）——`token=Bearer <凭证>` 这类「字段名 + 鉴权 scheme」形态，
`sensitive-field-kv` 的值取到空白为止，只会抹掉 `Bearer` 这个词；而独立的 `bearer` 规则排在
它之后，等它跑到时前缀已被替换、正则再也匹配不上，凭证本体于是全程无人处理。现在由
`sensitive-field-auth-scheme` 在 kv 规则**之前**把这种形态一路抹到行尾。新增规则时先想清楚
它与已有规则的先后关系：**先跑的规则做了局部替换，会让后面本该整段抹掉的规则失配。**

另一次坑（2026-08-06 review P1）——字段名被**整体引起来**时闭合引号夹在名与分隔符之间
（`{ 'x-api-key': 'opaque' }`、JSON `"x-api-key":"..."`），`auth-header` 要名后紧跟冒号、
`sensitive-field-json` 要引号紧贴敏感名（不容 `x-` 前缀）、`quoted`/`kv` 要名后紧跟分隔符，
四条全失配，于是带连字符/前缀的头（`x-api-key` 等）的值原样外泄。现由 `sensitive-field-quoted-key`
在其它字段规则**之前**吃掉这种「引号包起来的键」。**别按厂商加 key 形态就以为覆盖了头部字段——
字段名的书写形态（裸 / JSON / 转义 JSON / 带引号的对象键）同样要逐个想到。**

## 3. 授权

- 手动上传：要求已配置上报目标 **且** 已明示同意《隐私政策》。点击按钮本身即用户对这一次
  上传的意图，**不看「使用统计」开关**——那是行为埋点的偏好，与排障上传不是一件事。
- 自动上传（崩溃）：额外要求用户显式开启「崩溃时自动上传」，该开关**默认关闭**。
- **授权判定必须现读盘**：开发版与正式版共享 userData，用户可能在另一个实例里刚刚撤回授权。
  判定前调 `refreshFromDisk()`（mtime 守卫，文件没变时零开销）。
- 授权状态**读不出来时是 `unknown`，不是 `denied`**：不上传，但保留标记，把最终判定留给下次
  启动的可靠读取。用一次读取失败永久丢掉一个崩溃现场是不可接受的。
  ⚠️ 这条有个反直觉的实现要求：`createOverrideSettingsFile` 读到坏 JSON **会吞掉异常并返回
  默认值**，所以「文件损坏」与「用户明确没同意 / 明确关掉开关」在返回值上一模一样。闸的
  `readPrivacyConsentAccepted` / `readCrashAutoUploadEnabled` 因此必须先用
  `isAnalyticsConsentRecordReadable()` / `isLogUploadSettingsReadable()` 探一次，探到「文件在但
  解析不出来」时**抛异常**——只返回 `false` 会让闸判成 denied 并清空待补传标记。
  这两个探针取「现读盘」与「启动期 probe 结论」的交集：store 读到坏文件时会把它删掉，
  只现读就看不到了（设置页挂载即读一次，远早于延迟执行的启动补传，所以「启动期 probe」
  这一半不是可选项）。
- **反过来也要成立：手动路径不读崩溃开关。** 上面那条「读不出来 ⇒ unknown」如果作用到手动
  路径，就意味着「崩溃自动上传偏好文件损坏」会把用户主动点的上传一起堵掉（`unknown` 在 IPC
  层映射成 `PRIVACY_CONSENT_REQUIRED`）——恰好是用户最需要交日志的时候。手动路径的放行条件里
  本来就没有这个开关，所以 `evaluateGate` 在 `reason === 'manual'` 时**连读都不读**
  （2026-08-04 review P2）。一般化的口径：**闸只读该路径判定真正需要的状态**，多读一份就多
  一份把无关故障传染成拒绝的机会。
- 不做「上传前逐次弹窗确认」：手动路径本身就是用户点的；崩溃路径弹窗在崩溃时刻不可能可靠展示。

## 4. 上报目标与区域

目标 = 日志服务的 project + logstore + 服务区域接入域名，**构建期注入、运行期只读**。

链路（与 `config/endpoint.dev.json` / `release-regions.json` 同款约定）：

```
config/log-upload.json                     ← 真值,主仓 gitignore;唯一事实源在 cindy-build-scripts
  ↓ 打包机 sync-desktop-release-kit.sh 拷回
scripts/shared/log-upload-build-env.mjs    ← 全量校验 + 挑出「本构建区域那一个」目标
  ↓ apps/desktop/scripts/package-desktop.mjs 塞进 forge env
apps/desktop/vite.main.config.ts           ← define 成 main-only process.env.XDT_LOG_UPLOAD_TARGET
  ↓
main/log-upload/logUploadTarget.ts         ← 解析 + 区域交叉校验;不合法一律 null
```

- **仍然不进端点清单**（改成注入没有放松这一条）。两条独立理由：① 第三方域名不在
  `REGION_ENDPOINT_DOMAIN` 内，加进去会让整份离线缓存被判不可信、连带影响离线启动出口；
  ② 免签写入地址一旦可被**远程**改写，等于允许把用户日志改投他人的 logstore。构建期注入保留
  了「信任锚点不可远程改」这条性质，运行期配置不行。
- **只烘焙一个区域的目标**：cn 包里物理上不含 global 的 logstore 地址，反之亦然。这比「包里带
  两份、运行时按 region 选」更强——后者只要选错一次就写到另一区去了，而那正是有先例的事故
  （埋点曾因 global 构建报进国内采集端，导致国际项目缺失全部客户端数据）。
- **区域交叉校验**：注入串里带 `region`，运行时必须与烘焙的 `VITE