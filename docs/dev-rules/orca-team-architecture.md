# Orca 协同架构与执行单元规划

> 这是 Orca 的唯一权威文档，覆盖当前实现、设计约束与未来规划。
>
> **状态**：权威（authoritative）。
> **约束力**：本文「协同运行时行为契约」「坑点与不变量」两节是 Orca 协同代码的约束源，治理范围包括 `apps/desktop` 的 `maker-ipc/orca*` 服务与 `mcp-integrations` codex MCP 集成、`packages/lizi-mcps` 的 `orca` server 与 worker 控制工具、`packages/orca-workflow`、`packages/maker-core` 的 codex MCP context。改这些模块前先读本文；本文与代码实现冲突时以代码为准（遵循「以源码实现为准」原则），但必须同步修正本文。

## 目录

- [Part 1 · 当前架构](#part-1--当前架构)
  - [目标与边界](#目标与边界)
  - [核心概念与数据模型](#核心概念与数据模型)
  - [PR #101 后的服务边界](#pr-101-后的服务边界)
  - [MCP 与 IPC 控制面](#mcp-与-ipc-控制面)
  - [Codex Lead 机制](#codex-lead-机制)
  - [UI、Split View 与 Worktree](#uisplit-view-与-worktree)
  - [协同运行时行为契约](#协同运行时行为契约)
  - [测试与回归清单](#测试与回归清单)
  - [坑点与不变量](#坑点与不变量)
- [Part 2 · 未来规划](#part-2--未来规划)
  - [执行单元愿景](#执行单元愿景)
  - [三态拆分](#三态拆分)
  - [Worker 生命周期分型](#worker-生命周期分型)
  - [顶层类型与 Runner 分层](#顶层类型与-runner-分层)
  - [Side Chat](#side-chat)
  - [关系契约](#关系契约)
  - [控制面九大机制](#控制面九大机制)
  - [分阶段路线](#分阶段路线)
  - [待讨论项](#待讨论项)

## Part 1 · 当前架构

### 目标与边界

Orca 是 Cindy Desktop 内的多 agent 协同能力：一个 **Lead session** 负责拆任务、派活、验收与汇总，多个 **Worker session** 负责并行或串行执行。Worker 是完整会话，不是一次性 subagent；它有自己的模型、effort、工具调用流、上下文与可见历史。

当前已经实现的能力：

- 多 worker：同一个 active team 下可创建多个 worker，支持 role、label、focused worker 切换、soft/hard limit 与归档。
- Split view：Lead 与 focused Worker 共用 `OrcaSplitView` pane 外壳，宽屏为左右 split，doc rail 为 Lead/Worker toggle。见 `apps/desktop/src/renderer/features/cc-agent/OrcaSplitView.tsx` 的 `OrcaSplitView`、`OrcaPaneShell`。
- 本地 Claude Code、Codex 与 Pi 的普通 Lead 无论是项目还是对话都可开启协同，本地 Worker 也支持这三类 agent；Pi 当前仅支持本地 Lead / Worker，不支持 SSH 远程协同。SSH 远端会话的 Claude Code 与 Codex 两端均可作 Lead（远端 agent 经 SSH remote-forward 直连本机 HTTP MCP bridge，`cindy_orca` 在两端都可用：codex 走 daemon config 注入，cc 走 per-query http 注入）；device-link 被控端的项目与对话同样可作 Lead（Lead / Worker / team 的真身都在被控端，控制端只是镜像）。renderer 的入口判定收敛在 `apps/desktop/src/renderer/features/cc-agent/collabEntryPolicy.ts` 的 `resolveCollabEntryPolicy`，新建草稿（`NewMakerDraftRoute`）与会话视图（`CCAgentSessionView` 的 `allowCollabToggle`）共用同一份：普通 Lead 都显示入口，只排除不能嵌套协同的 Orca Worker 子会话。项目按项目级策略查询；Main 的 `resolveLocalCollabPolicyWorkingDir` 同时服务入口状态查询与最终授权，只有 workspace kind 为 dialogue 且 Main 确认目录位于 app 托管 dialogue root 时才只查用户级/全局级策略（即使 cwd 内出现 `.cindy/plugins.json`），显式绑定真实目录的对话仍按该目录的项目级策略查询，不能靠自报 `workspaceKind` 绕过项目禁用。
- Codex Lead 使用全局注册的 `cindy_orca`，调用时通过 context 恢复身份并在 handler 内拒绝越权。远端 Codex 同样走 `params._meta.threadId` 路由——remote thread 与本地 thread 一样注册进 `CodexMcpThreadContextStore`（`packages/maker-core/src/agents/codex/index.ts` 的 `registerCodexMcpContext` 不再跳过 remoteHostId）。远端 cc 没有 threadId，身份走持久 bearer token + URL `?session=<id>` 路由（`codexHttpBridge.registerSessionCtx`），审批归属快照在 `remoteCcQueryFactory` 注入后按 `startParams.mcpServers` 最终清单定稿。
- SSH 远端 Codex 的 MCP 桥接：本机 `codexHttpBridge` 在原有 per-run 主 token 之外接受一个 persistent bearer token（safeStorage）；`remote-ssh/codex-remote-mcp.ts` 在 session start/resume 前置完成 per-host 固定端口 remote-forward（`RemoteHost.openRemoteForward`，重连自动 rebind）、远端 `$CODEX_HOME/config.toml` 的 `mcp_servers` 管理段漂移检测（行级 marker + 剥离用户同名 table）与 daemon 幂等 bootstrap（token 只经 stdin 的 KEY=value 块注入，不进 argv）；config 漂移需要重启 daemon 时若同 host 有 live turn 则本次降级（留待下次 ensure）。worker 创建经 `OrcaLeadSessionSnapshot.remoteHostId` 继承在同一台远端主机 spawn，创建前走 `ensureRemoteReadyForSessionStart`（SSH 重连 / agent 安装 / MCP 注入）；lazy resume 与 Orca worker 唤醒路径同样先 ensure 再 bootstrap。
- PR #107 提供 side_chat 的底层 fork 数据动作：user/assistant 消息都能 fork，assistant 按 turn 粒度复制，Claude 用 uuid 锚点，Codex 用 ThreadFork + ThreadRollback。当前作为普通 session 跳转；未来要登记为 side activity 并挂入统一 pane。

当前边界：

- Orca Team 仍严格绑定单个 Lead session，不支持跨 Lead 共享 worker。
- Worker focused 切换是“单 worker pane 替换”，不是多 worker 并排。
- Worker archive 后 UI 内没有 unarchive 入口；DB 记录保留，用户需要新建 worker 继续。
- workflow_run / CC Workflow 编排还未纳入当前实现。
- side_chat 尚未登记为 side activity 对象，也未挂进 pane；PR #107 只是 fork 数据动作。
- device-link 协同的 Lead / Worker / team 全部在被控端进程内编排，控制端只按 session 来源经隧道路由（`makerTransport` 的 `makerApiFor` / `orcaWorkflowsFor` / `subscribeOrcaWorkerChanged`，channel 见 `packages/device-link/src/allowlist.ts` 的 Orca 段）。collab 开关同样查被控端（`maker:plugins:get-state` 经 `pluginEnableStateFor`）：项目读取被控端项目级策略，对话读取被控端用户级/全局级策略。控制端本机状态不能代表被控端真相。老被控端没有该 channel 时回 `CHANNEL_NOT_ALLOWED`，控制端 fail-closed 置灰入口并提示设备版本过旧，而不是放行到 `enableOrca` 才撞错。
- SSH 远端协同仅支持 codex 与 claude-code 两类 lead + 远端 worker（继承 remoteHostId），不支持 Pi Lead 或 Pi Worker。cc 远端经 `cc-remote-mcp.ts` 把 `cindy_orca` / `orca_worker_bridge` 以 http 形态追加进 `startParams.mcpServers`（persistent token + `?session=` 路由，白名单仅此两个 server）。远端 worker 手动 `send_to_lead` 依赖 daemon 侧 `orca_worker_bridge` 经同一 bridge 可达；auto-bridge 回报不依赖 worker 侧 MCP，天然可用。共享 userData 多实例连同一远端 host 时，只有先建立 SSH 转发的实例能持有该 host 的 MCP bridge 端口，其余实例按“远端无 MCP”降级（与历史行为一致）。远端会话的项目级 collab 开关不查本机 fs；`assertCollabProjectEnabled` 对 remote 只查用户级/全局级开关，远端项目级配置机制是 follow-up。

### 核心概念与数据模型

| 概念 | 当前含义 | 载体 |
|---|---|---|
| Lead session | 主线会话，负责开 team、创建 worker、派活与验收 | `sessions.orca_role = 'lead'` |
| Worker session | 被 Lead 派出的完整 side session，有独立 agent/model/context | `sessions.orca_role = 'worker'` + `orca_workers.session_id` |
| Team | 一个 Lead 当前 active 的协同上下文 | `orca_teams` |
| Worker record | worker 在 team 内的 role、label、status、focused 等元数据 | `orca_workers` |
| Fork relation | 普通 session fork 来源关系，为 side_chat 底座 | `sessions.parent_session_id` + `forked_at_message_id` |

`orcaRole` 和 fork parent 是两个字段，不能混用。schema 明确写着 `orca_role` 用于 Orca split-session role，`parent_session_id` 保留给 fork/session-branch，见 `apps/desktop/src/main/localDb/schema.ts` 的 `sessions` schema。fork 关系不能承载 Orca 派单语义，这是后续把 side_chat 纳入 side activity 的数据前提。

`orca_teams` 保证同一个 Lead 同时最多一个 active team：唯一约束 `uniq_active_team_per_lead` 只约束 `status = 'active'`。约束：active 唯一性必须用 partial unique，不能用全表 unique，否则非 active team 占住索引会让同一个 Lead 无法重新开启协同。

`orca_workers` 保存 team 内 worker：`team_id`、`session_id`、`status(idle/running/done/error)`、`label`、`role`、`focused`、`idle_since`。同一个 team 只能有一个 focused worker，靠 partial unique `uniq_orca_workers_focused_per_team` 保证，见 `apps/desktop/src/main/localDb/schema.ts` 的 `orcaWorkers` schema。

fork/session-branch 的最小 provenance 当前只有两列：

- `parent_session_id`：源 session self-FK，源删掉时 set null。
- `forked_at_message_id`：来源 message 的 client id，无 FK。PR #107 后 assistant fork 也会写 assistant client id；schema 注释需要按实际语义同步，见 `apps/desktop/src/main/localDb/schema.ts` 的 `sessions.forkedAtMessageId`。

### PR #101 后的服务边界

PR #101 之后，Orca 的 main 侧业务由独立 service 承接，`register.ts` 只负责依赖装配、IPC/MCP wiring 和 host adapter。服务边界如下：

| 服务 | 文件 | 责任 |
|---|---|---|
| `OrcaLifecycleService` | `apps/desktop/src/main/maker-ipc/orcaLifecycleService.ts` | `start_team`、开启协同 `enableTeam`、创建 team、读取或更新 Worker 创建权限偏好、设置 Lead `orcaRole`、首个 worker 创建补偿 |
| `OrcaWorkerCreationService` | `apps/desktop/src/main/maker-ipc/orcaWorkerCreationService.ts` | 既有 team 下创建 worker，统一 role/label/model/effort/fast 校验与默认值；新 Worker 使用 `workerCreationPrefs` 的权限偏好，不继承 Lead 当前模式；创建 worker session 并写 `orca_workers` |
| `OrcaTeamService` | `apps/desktop/src/main/maker-ipc/orcaTeamService.ts` | 给既有 worker 派活、resume、idle、archive、terminal turn 处理与 auto-bridge |
| `OrcaInterAgentDispatcher` | `apps/desktop/src/main/maker-ipc/orcaInterAgentDispatcher.ts` | Lead/Worker 之间的消息直发或排队、accepted callback、rollback/settle 语义 |

契约：team lifecycle、worker creation、worker lifecycle/auto-bridge、inter-agent dispatch 四类职责不能互相偷跑；所有入口都必须复用这组 service，不能在 IPC 或 MCP handler 内各自实现一套状态机。关键实现入口是 `OrcaLifecycleService.startTeam/enableTeam`、`OrcaWorkerCreationService.createWorkerInTeam`、`OrcaTeamService.dispatchWorkerTask/handleWorkerTerminalTurn`、`OrcaInterAgentDispatcher.dispatchOrEnqueueOrcaInterAgentMessage`。

### MCP 与 IPC 控制面

`cindy_orca` 是独立 MCP server，直接顶层注册 16 个工具：13 个 team 控制工具 + 3 个只读诊断工具（后者从已下线的 `orca_bridge` 桥入，保裸名）。它与 renderer IPC 共用同一组 main 侧 service，因此 MCP 和 UI 操作必须有一致的权限、状态和回滚语义。

1. `start_team`
2. `end_team`
3. `create_worker`
4. `create_workers`（批量创建；顺序执行、hard limit 后停止并返回逐项汇总）
5. `list_workers`
6. `switch_focus`
7. `send_to_worker`
8. `list_worker_queue`（排队消息控制）
9. `update_queued_message`（排队消息控制）
10. `cancel_queued_message`（排队消息控制）
11. `idle_worker`
12. `archive_worker`
13. `list_available_models`
14. `get_workspace_info`（只读诊断）
15. `worker_status`（只读诊断）
16. `read_worker`（只读诊断）

批量创建必须走一次 `create_workers` 调用，不能让 Lead 并行或连续发多个独立 `create_worker`。批量工具按输入顺序复用同一个 `OrcaLifecycleService.createWorker` 原语；首次收到 `WORKER_LIMIT_HARD_EXCEEDED` 或批次级 `HOST_NOT_READY` 后不再调用 host，剩余项稳定标为 `skipped`。返回值必须包含请求数、实际尝试数、成功数、失败数、跳过数、总未创建数、数量闸快照、代码确定生成的 `user_report`，以及逐项真实 worker/session 或失败终态，供 Lead 如实向用户收口；`success_count + failure_count + skipped_count` 必须等于 `request_count`，其中 `not_created_count = failure_count + skipped_count`。单个 `create_worker` 继续作为兼容入口，并返回相同的结构化 hard-limit 快照。

排队消息控制 3 工具让 Lead 在消息被 worker 消费前管理自己发出的排队消息：`send_to_worker` / `create_worker`（initial_task）在 `wakeKind='queued'` 时回传 `queued_message_id`（coordinator 队列内的 clientId），Lead 可据此列出、整条改写或撤回。实现走 `OrcaTeamService.listWorkerQueuedMessages / updateWorkerQueuedMessage / cancelWorkerQueuedMessage`，语义约束见「协同运行时行为契约 · 消息派发与 auto-bridge」第 6 条。

诊断 3 工具是纯只读，实现走 host `apps/desktop/src/main/maker-ipc/orcaDiagnostics.ts`（读 active team + DB worker 列表 + live session 状态 + 最近 assistant 消息），**无建 team 写副作用**——这是与旧 `orca_bridge` 版本的关键差异（旧版经 `ensureWorkflowForLead` 会顺手建 team）。早期 Lead 侧 `orca_bridge`（门面 + 私有 registry/restore/auto-bridge）已整体删除：Lead→worker/team 工具面唯 `cindy_orca`（C）一套，worker→lead 唯 `orca_worker_bridge`（B，在 `packages/orca-workflow`）一套；`@cindy/orca-workflow` 包通过 B provider 与 `renderOrcaLeadSystemPrompt`/`renderOrcaWorkerSystemPrompt` 两个 render 函数继续被 host 依赖。

`cindy_orca` 直接注册到顶层，而不是藏在 `list_tools/call_tool` 后面；模型在“开协同 / 派 worker”时需要稳定发现 `start_team/create_worker`。实现见 `packages/lizi-mcps/src/orca/server.ts` 的 `createOrcaMcpServer`、`DirectToolSink`、`OrcaMcpDeps`。

Worker 权限是 **Worker 创建偏好**，与 Agent、模型、effort、Fast 的“下次创建默认值”同类，不是 Lead 权限的继承项，也不是 Team 数据库字段：

- 真源是 renderer `workerCreationPrefs` localStorage；main 只保存内存镜像，应用启动和偏好变化时由 renderer 同步。
- 没有保存过权限偏好时，产品默认是 `bypassPermissions`（Full access）；这只是可选择的初始值，不是固定模式。UI 每次创建 Worker 都可改选 `auto` 或 `bypassPermissions`，提交后成为下一次默认值；已经保存过的选择继续优先，不随产品默认值变化而改写。
- M