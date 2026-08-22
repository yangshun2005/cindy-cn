# 客户端日志上报（Desktop）实现方案

> 状态：**已落地**（P0 + P1 已实现；cn / global 上报目标已配置，dev 待配）
> 　最后更新：2026-08-04　范围：`apps/desktop`
>
> 配套需求文档：[`client-log-upload-requirements.md`](client-log-upload-requirements.md)。
> 落地后的**开发规则**（改动前必读的三条不变量）已固定在
> [`dev-rules/log-upload-and-redaction.md`](dev-rules/log-upload-and-redaction.md)。
> 本文只讲**怎么落地**：模块边界、关键算法、对既有代码的改动点、测试与分期。需求
> 本身的取舍不在这里重复论证。
>
> **实现与本文的差异**（落地过程中发现的必要修正，已同步到代码与规则文档）：
> - `LOG_RETENTION_DAYS` 移到 `shared/logRetention.ts`，而不是从 `logger.ts` 导出——采集
>   管道刻意不 import logger（logger 依赖 electron，管道要能在纯 Node 单测里直跑）。
> - 新增 `MAX_BYTES_TOTAL`（全部文件合计 24 MiB）。只有 per-file 闸时，30 天窗口的最坏总量
>   会到几百 MB；文件按「与崩溃锚点的接近程度」排序后依次读，预算耗尽即停，被砍掉的一定是
>   离崩溃最远的那些天。
> - 环境元数据同时写进 `__tags__` **与每条记录的 `uploadCode` 字段**：`__tags__` 的索引配置
>   属服务端侧，客户端不该依赖一个自己看不到的配置才能被检索到。
> - `PendingMarkerStore.clearAll()` 必须同时清 claim 文件（claim 文件名不以 `.json` 结尾）。
>   只按未认领标记清的话，用户关闭授权时正在被本进程补传的那条会被漏掉、补传照常完成——
>   正是需求 §4.3 末条要禁止的行为。这条是写测试时发现的。

---

## 0. 现状核对（本方案的事实基础）

写方案前逐项核对了需求引用的既有实现，结论如下（与需求描述一致，无需返工）：

| 需求引用 | 代码事实 |
|---|---|
| 本地日志三流 + 30 天保留 | `apps/desktop/src/main/logger.ts`：`main-<date>.log`（纯文本）、`agent-<date>.ndjson`、`sessions/<id>/<date>.ndjson`，`LOG_RETENTION_DAYS = 30` |
| main 日志行格式 | `[${localTimestamp}] [${LEVEL_TAG}] [${scope}] ${msg}\n`，`LEVEL_TAG` 含 `INFO ` / `WARN ` 的补位空格 |
| 渲染进程日志天然带前缀 | `writeFromRenderer()` 强制 `r:` 前缀 |
| 崩溃转储只落本地 | `startup-diagnostics.ts`：`crashReporter.start({ uploadToServer: false })` |
| 退出尸检结论只进日志 | `RunMarkerStore.analyzePreviousRuns()` → `logPreviousRunReports()`，`reports` 未对外暴露 |
| 致命判定收口在生命周期模块 | `lifecycle.ts` 的 `beginShutdown(timeoutMs, reason)`；`unhandledRejection` / broken stdio / 瞬时网络错误都**不**进入该路径 |
| 端点清单有受信任域约束 | `endpointManifestCache.ts` 的 `REGION_ENDPOINT_DOMAIN` + `findUntrustedCachedEndpoint()`——第三方日志服务域名加进清单会让整份离线缓存判不可信 |
| 隐私政策同意是独立事实 | `analytics-settings-store.ts` 的 `privacyConsentAccepted`（与 `analyticsEnabled` 分开） |
| 跨实例现读盘的现成原语 | `maker-host/override-settings-file.ts` 的 `invalidateIfChanged()`（mtime 守卫，文件没变零开销） |
| 吃系统代理的出网通道 | `maker-host/outbound-fetch.ts` 的 `outboundFetch` |
| 区域常量 | `shared/brandRegion.ts` 的 `CURRENT_CINDY_REGION`（main / renderer 共用） |
| 设备与用户标识 | `authManager.getDeviceId()`、`authManager.getAuthState().user?.id` |
| 设置页落点 | `renderer/components/settings/AboutSection.tsx` 的 `OpenLogsRow` 之后；开关的「已修改 / 恢复默认」走 `DefaultOverrideControls` |

一处需求没点明、但实现必须处理的事实：**`main-*.log` 目前写多行 `msg` 时不做任何转义**
（`emit()` 直接把 `util.format` 结果塞进行内）。这正是需求 §5.5 那条不变量的缺口，见 §4.1。

---

## 1. 模块布局

新增一个自治目录，**只有 `index.ts` 碰 Electron**，其余全部纯逻辑 + 依赖注入（工程规范 §3）。

```
apps/desktop/src/shared/
  logUpload.ts                  # IPC 契约类型、广播 channel、上传编号格式（main/preload/renderer 共用）
  mainLogRecordFormat.ts        # main 日志「记录边界」的唯一事实源：head 正则 + 续行转义函数
                                #   ↑ 写侧(logger.ts)与读侧(解析器)共用同一份，不允许各写一份

apps/desktop/src/main/log-upload/
  index.ts                      # 唯一的 electron 接线层：IPC 注册、启动补传调度、崩溃钩子接线
  logUploadTarget.ts            # 按区域烘焙的上报目标（穷举 Record<CindyRegion, …|null>）
  logUploadSettingsStore.ts     # 「崩溃时自动上传」开关（override 语义）
  consentGate.ts                # 授权闸：现读盘 + 三种 kind 的判定
  sourceAllowlist.ts            # 第二层：来源白名单（逐条带理由）+ 子来源排除
  redact.ts                     # 第三层：正则红线
  mainLogReader.ts              # main-*.log → 结构化记录（记录边界识别 + 定位读取）
  agentLogReader.ts             # agent-*.ndjson → 仅 proxy 记录
  collect.ts                    # 采集编排：源白名单 → 窗口 → 锚点裁剪 → 四层管道
  pendingMarkers.ts             # 待补传标记：代次 token + 原子认领 / 原子清除
  logSink.ts                    # 免签写入客户端（批次切分、超时、结果语义）
  uploadRunner.ts               # 一次上报的完整编排（闸 → 采集 → 发送 → 标记收尾）
  crashTriggers.ts              # 崩溃即时 / 启动尸检兜底的判定与标记写入
  __tests__/                    # 见 §7
```

依赖方向：`log-upload/*`（除 index）**不 import electron、不 import logger**，路径 / 时钟 /
fetch / 授权读取全部注入。`index.ts` 反向依赖 `lifecycle`、`startup-diagnostics`、
`authManager`、`analytics-settings-store`、`outbound-fetch`。无循环（`lifecycle` 与
`startup-diagnostics` 只被 index 调用，反向靠回调注册，见 §4.2 / §4.3）。

---

## 2. 上报目标与区域绑定（需求 §4.4）

### 2.1 配置形状

```ts
// shared/logUpload.ts
export interface LogUploadTarget {
  /** 日志服务 project */
  project: string;
  /** logstore */
  logstore: string;
  /** 服务区域接入域名（不含协议、不含 project 前缀） */
  endpointHost: string;
}
```

**值不写在代码里，走构建期注入**（2026-08-04 owner 选定方案 B），与
`config/endpoint.dev.json` / `apps/desktop/scripts/release-regions.json` 同款约定：

```
config/log-upload.json                     ← 真值,主仓 gitignore;唯一事实源在 cindy-build-scripts
  ↓ 打包机 sync-desktop-release-kit.sh 拷回
scripts/shared/log-upload-build-env.mjs    ← 全量校验 + 挑出「本构建区域那一个」目标
  ↓ apps/desktop/scripts/package-desktop.mjs 塞进 forge env
apps/desktop/vite.main.config.ts           ← define 成 main-only process.env.XDT_LOG_UPLOAD_TARGET
  ↓
main/log-upload/logUploadTarget.ts         ← 解析 + 区域交叉校验;不合法一律 null
```

配置文件形状（仓内 `config/log-upload.json.example` 是可直接改值的骨架）：

```json
{
  "schemaVersion": 1,
  "cn":     { "project": "cindy-sh-prod",  "logstore": "cindy-sh-prod-client-log",  "slsRegion": "cn-shanghai" },
  "global": { "project": "cindy-sgp-prod", "logstore": "cindy-sgp-prod-client-log", "slsRegion": "ap-southeast-1" },
  "dev": null
}
```

实际写入地址：

```
cn      https://cindy-sh-prod.cn-shanghai.log.aliyuncs.com/logstores/cindy-sh-prod-client-log/track
global  https://cindy-sgp-prod.ap-southeast-1.log.aliyuncs.com/logstores/cindy-sgp-prod-client-log/track
```

关键设计点：

- **只烘焙一个区域的目标**。cn 包里物理上不含 global 的 logstore 地址，反之亦然——比「包里带
  两份、运行时按 region 选」更强，后者只要选错一次就写到另一区去了（埋点有过这个事故）。
- **区域交叉校验**。注入串带 `region`，运行时必须与烘焙的 `VITE_CINDY_AUTH_REGION` 一致。
  这是「注入链路串了」（打包机 env 残留、本地 `.env` 放了另一区的目标）唯一的运行期防线。
- **仍然不进端点清单**：第三方域名不在 `REGION_ENDPOINT_DOMAIN` 内，加进去会让整份离线缓存判
  不可信；且免签写入地址可被**远程**改写 = 允许把用户日志改投他人 logstore。构建期注入保留了
  「信任锚点不可远程改」这条性质。
- ⚠️ **fail-closed 的位置从 typecheck 搬到了构建脚本**。原先 `Record<CindyRegion, …>` 让「新增
  区域忘了做选择」直接编译失败；值搬进 gitignore 的 JSON 后由 `log-upload-build-env.mjs` 的硬
  校验替代：除 `dev` 外每个区域必填（缺失/非法即打包失败）、两区不得共用 project 或 logstore、
  `slsRegion` 只写区域代号。新增发行区域自动成为必填，不需要有人记得改校验名单。
- 目标解析为 null ⇒ `targetConfigured: false` ⇒ 手动入口禁用并给出「未配置」提示、自动路径
  **一个字节都不发、也不写标记**。dev server / 本地 `pnpm dev` 拿不到注入值 ⇒ 天然关闭。
- **代价**：真值不进仓 ⇒ CI 看不到它 ⇒ desktop 侧单测只锁形状（解析、区域校验、URL 模板），
  真值正确性由 `scripts/__tests__/log-upload-build-env.test.mjs`（对 `.example` + 临时 fixture）
  加打包硬失败兜底。这是选方案 B 换来运维不碰主仓的对价。

### 2.2 免签写入

Web-tracking 形态（无 AccessKey，需求 §5.3）：

```
POST https://<project>.<endpointHost>/logstores/<logstore>/track
Content-Type: application/json
x-log-apiversion: 0.6.0

{ "__topic__": "client-log",
  "__source__": "<platform>-<arch>",
  "__tags__":  { …环境元数据(§6)… },
  "__logs__":  [ { "ts": "...", "level": "...", "src": "...", "scope": "...", "msg": "..." }, … ] }
```

- 走 `outboundFetch`（吃系统代理），每批 20s 超时，批次**串行**发送，任一批非 2xx ⇒
  整次判失败（标记保留）。
- 批上限：`MAX_LOGS_PER_BATCH = 500`、`MAX_BATCH_BYTES = 1 MiB`（留足服务端 3MB 余量）。
- 时间戳：web-tracking 的 `__time__` 由服务端接收时刻决定，**因此原始时间戳一律作为
  普通字段 `ts` 携带**（需求 §4.8「不能用上报时刻覆盖」由此满足），不依赖 `__time__`。
- `logSink.ts` 只依赖注入的 `fetchImpl`，单测用内存 fake 断言 URL / 批次切分 / 失败语义。

**需要向日志服务 owner 确认（合并前）**：`__tags__` 是否默认建索引（决定 `uploadCode` 是否
需要同时写进每条记录）；logstore 是否已开启 web tracking；留存期与访问权限（需求 §9.4）。

---

## 3. 采集与脱敏管道（需求 §4.2，本方案的核心）

`collect.ts` 的签名（全注入，零 electron）：

```ts
export interface CollectDeps {
  logDir: string;
  listDir(dir: string): Promise<string[]>;
  openRead(file: string): Promise<LineSource>;  // 只暴露「按 offset 流式读行」能力
  now(): number;
  homeDir: string;
  yieldToEventLoop(): Promise<void>;            // 每 N 行让出一次,不霸占 main 事件循环
}
export interface CollectRequest {
  reason: 'manual' | 'crash-immediate' | 'crash-backfill';
  /** 崩溃锚点(epoch ms),manual 为空数组 */
  anchors: number[];
}
export interface CollectResult { records: UploadRecord[]; stats: CollectStats; }
```

### 3.1 第一层：源白名单（读哪些文件）

- 主体：`main-<date>.log`，`date` 落在时间窗内。
- `reason !== 'manual'` 时附带：logs 根的 `agent-<date>.ndjson`，且**只取
  `source === 'proxy'` 的记录**（同一文件里还有 `maker` 源的启动期日志，可能带用户内容，
  逐条丢弃）。
- `sessions/**` 与 `*cc-debug.raw.log` **永不构造路径**。这条靠 `openRead` 注入 + 测试锁定：
  测试在 fixture 里放好诱饵文件，断言被打开的路径集合不含它们（需求 §6 隐私性第 1 条）。

### 3.2 时间窗与「定位读取」

```
MAX_LOOKBACK_DAYS_DEFAULT = 2
MAX_LOOKBACK_DAYS_CAP     = LOG_RETENTION_DAYS  // 从 logger.ts 导出复用,不写第二份 30
MAX_BYTES_PER_FILE        = 8 MiB
MAX_RECORDS               = 4000
MAX_MSG_CHARS             = 2000
```

窗口 = `max(MAX_LOOKBACK_DAYS_DEFAULT, 距最早锚点的天数)`，再 clamp 到 `MAX_LOOKBACK_DAYS_CAP`；
超出保留期的锚点直接放弃（日志已被清理，补传无意义，需求 §4.5）。

单文件超过字节预算时**不能简单读尾部**——崩溃后堆积的新日志会把崩溃现场挤出窗口
（需求 §11 的既有教训）。做法：main 日志按天单文件内时间**严格递增**且每条记录首行自带
可解析时间戳，因此可以对文件做**时间戳二分查找**，定位到「最早锚点 - 预卷（默认 2 min）」
的字节偏移，从那里起读 `MAX_BYTES_PER_FILE`。manual（无锚点）退化为读尾部。

### 3.3 记录边界识别

`shared/mainLogRecordFormat.ts` 是唯一事实源：

```ts
export const MAIN_LOG_RECORD_HEAD_RE =
  /^\[\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}\.\d{3}[+-]\d{2}:\d{2}\] \[(TRACE|DEBUG|INFO |WARN |ERROR|FATAL)\] \[([^\]]*)\] /;

/** 写侧:把 msg 的每个续行前置一个空格,保证「除首行外没有行以边界特征开头」。 */
export function escapeMainLogContinuationLines(msg: string): string;
```

读侧按 head 正则切记录，命中即新记录，否则并入上一条记录的续行。

### 3.4 第二层：来源白名单（deny-by-default）

`sourceAllowlist.ts` 是一张**逐条带理由**的表，匹配方式为根锚定：
`scope === root || scope.startsWith(root + '/') || scope.startsWith(root + ':')`。
因此 `r:` 前缀的渲染进程日志整类落空（需求 §4.2），`maker*` / `cc-proxy*` / `codex-proxy*`
也不会命中（它们本就不写 main 流）。

初始放行名单（按需求要求的六类基础设施划分；每条在代码里带一句理由注释）：

| 类别 | 放行根 scope |
|---|---|
| 生命周期 / 崩溃 / 进程 | `lifecycle`、`startup-diagnostics`、`logger`、`process`、`power-diagnostics`、`power-blocker`、`idleManager`、`app-presence`、`appSessionState`、`renderer-gu