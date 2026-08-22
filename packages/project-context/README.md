# project-context

Agent 维护的项目知识层。基于提交驱动，使用 Markdown + frontmatter 格式，提供通用 CLI。

本文档同时维护使用方式、架构边界与当前限制。

> **状态**：MVP（v0.1）。MVP 范围见本文档底部“限制”部分。

---

## 这是什么

每次开启新的 agent 会话时，agent 都需要重新搜索项目结构来理解代码。`project-context` 将这一层“项目知识”持久化下来 —— 跟随 git 提交增量演进，**由 agent 自己维护**，并在新会话启动时按需注入。

核心概念：

- **模块**：与包/目录对齐的骨架知识（每个包对应一份 .md 文件）
- **关注点**：跨包的横向关注（IPC、agent 生命周期等，由 agent 在更新时提议；MVP 阶段不自动创建）
- **manifest.yaml**：索引文件，用于反向查询“哪些知识覆盖某文件”
- **过期标记**：知识过期时打上标记，等下次任务用到时按需刷新（JIT，第二阶段）
- **自动更新标志**：在 frontmatter 中设置 `auto_update: false` 可冻结某个模块，使其不被自动维护

---

## 安装 / 运行方式

### 在本仓库内（开发态）

`project-context` 是 monorepo 内的 workspace 包，无需独立安装。直接：

```bash
# 1. 安装依赖（仓库根）
pnpm install

# 2. 编译
pnpm --filter project-context build

# 3. 运行 CLI（仓库根，使用绝对路径）
node packages/project-context/dist/cli.js <command>
```

### 通过 pnpm exec（可在任意 workspace 子目录运行）

```bash
pnpm --filter project-context exec project-context <command>
```

### 通过 npx（包发布或全局安装后）

未来发布到 npm 后：

```bash
# 全局安装
pnpm install -g project-context
project-context <command>

# 或按需使用 npx
npx project-context <command>
```

### 项目内 devDep（推荐的生产部署方式）

```bash
pnpm add -D project-context
# package.json 中就会有 bin shim：
node_modules/.bin/project-context <command>
# 或在 pnpm 脚本中：
# "scripts": { "kctx": "project-context update" }
```

### 开发模式（无需每次构建）

```bash
pnpm --filter project-context dev   # tsx watch src/cli.ts
```

> **依赖项**：Node ≥ 18，且**仓库必须是 git**。MVP 强制要求；非 git 项目将报错退出。

---

## 快速开始（3 步跑通）

```bash
# 1) 初始化：扫描 monorepo / pnpm workspace，为每个有源码的包创建一份空壳 .md
node packages/project-context/dist/cli.js init

# 2) 查看产出
ls .cindy/project-knowledge/modules/
cat .cindy/project-knowledge/manifest.yaml

# 3) 模拟有 diff 时的试运行（不调用 LLM、只计算映射）
node packages/project-context/dist/cli.js update --since HEAD~5 --check-only

# 4) 反向查询“修改了某个文件会影响哪些知识”
node packages/project-context/dist/cli.js query --files packages/maker-core/src/agents/base-agent.ts
```

---

## 命令

### `init [--bootstrap] [--prune]`

扫描仓库，按包管理器约定创建 `modules/` 骨架。**幂等**：已存在的 .md 文件不会被覆盖。

发现优先级：
1. `.cindy/project-knowledge/config.yaml` 中的 `module_roots`（显式）
2. `pnpm-workspace.yaml` 中的 `packages` 字段
3. `package.json` 中的 `workspaces` 字段
4. 都找不到 → 报错并提示编写 config.yaml

每个候选目录都会经过**源码启发式检查**：目录中没有任何 `.ts/.js/.go/.rs/.py/.html/.css/...` 等源码后缀则跳过（用于排除 `*-bin` 这类纯二进制镜像包）。

**选项：**

| 标志 | 作用 |
|---|---|
| `--bootstrap` | 运行一次“全工程扫描 + LLM 提议跨切关注”。**MVP 占位，暂未真正运行**，会输出黄色提示 |
| `--prune` | 将孤立模块（发现过程不再能找到的 .md）从磁盘删除。默认只**报告**不删除 |

**典型输出：**

```
project-context init
  repo root: D:/work/cindy
  discovered modules: 8
  created: 8, skipped: 0
    + apps--desktop
    + apps--landing-page
    + ...
  wrote default .cindy/project-knowledge/config.yaml
  manifest: .cindy/project-knowledge/manifest.yaml
```

### `update [--since <ref>] [--check-only]`

根据 git diff 增量更新受影响的知识文件。

流程：
1. 获取文件锁（`.cindy/project-knowledge/.lock`），获取失败则跳过本次（防止并发）
2. 计算 `last_synced_commit..HEAD` 的 diff
3. 经过 `.gitignore` + 内置忽略规则 + config.ignore 三层过滤
4. 反向查询每个修改文件所覆盖的知识 ID
5. 对每个受影响的 ID：
   - **小 diff**（≤ 5 个文件且 ≤ 200 行，可在 config.yaml 中调整）→ 调用 agent 适配器重写正文
   - **大 diff** → 标记 `stale: true`，等待下次任务刷新
   - **`auto_update: false`** → 完全跳过（不重写也不标记 stale）
6. 收集“未被任何知识覆盖”的修改文件，提示考虑添加新模块/关注点

**选项：**

| 标志 | 作用 |
|---|---|
| `--since <ref>` | 覆盖 diff 起点。默认使用 manifest 中的 `last_synced_commit`。常用 `--since HEAD~5` 测试 |
| `--check-only` | 只计算不写入：列出将要更新 / 标记 stale / 只读跳过的 ID，不调用 LLM 也不修改文件 |

**典型输出：**

```
project-context update
  diff: 4cc6490..6838258
  updated: 1, staled: 1, read-only skipped: 0, uncovered files: 1
    ~ packages--maker-core      (rewrote)
    s apps--desktop             (staled)
  uncovered files (consider adding a module/concern):
    - .gitlab-ci.env
```

> 真正调用 LLM 需要本机 PATH 中有已登录的 `claude` CLI（详见适配器章节）。

### `refresh [ids...] [--all | --stale] [--check-only] [--force]`

**从当前源码重建知识正文**，不依赖 git diff。适用于冷启动 / 历史代码补全 / 修复过期标记。

适用场景：
- **首次冷启动**：`init` 只创建空骨架（TODO 占位），`refresh --all` 让 LLM 一次性填充所有模块
- **过期修复**：之前 update 因 diff 过大而标记了 stale，运行 `refresh --stale` 集中重新生成
- **单模块刷新**：只想查看一个模块的产出，运行 `refresh packages--maker-core`

范围三选一（互斥）：

| 用法 | 作用 |
|---|---|
| `refresh <id> [<id> ...]` | 刷新指定 ID（可多个） |
| `refresh --stale` | 仅刷新标记为 stale 的模块 |
| `refresh --all` | 全部刷新 |

**选项：**

| 标志 | 作用 |
|---|---|
| `--check-only` | 试运行：列出目标但不调用 LLM 也不写入磁盘 |
| `--force` | 默认跳过 `auto_update: false` 的模块；添加 `--force` 强制刷新 |

**机制**：refresh 通过适配器调用 `claude -p`，但**不提供 diff**。只告诉 agent：“你要刷新的模块覆盖 `<covers>` 路径，请用你的 Read/Glob/Grep 工具自行探索源码，然后编写新正文”。`claude` 在 headless `-p` 模式下默认具备工具访问权限，因此可以自主探索。

**典型输出：**

```
project-context refresh
  targets: 8 (scope=all)
  [1/8] apps--desktop  refreshing...
     done (32.4s)
  [2/8] apps--landing-page  refreshing...
     done (8.1s)
  ...

  refreshed: 7, skipped: 0, failed: 1
    x packages--@cindy/mcps  claude-code adapter: timed out after 600000ms
```

**成本提醒**：
- 单个模块刷新预计消耗 10-30k token + 约 30-90 秒（取决于包大小）
- 在 8 个模块的项目上，`refresh --all` 预计消耗 100-200k token + 5-15 分钟
- 强烈建议**先试运行一个模块**（`refresh packages--maker-core --check-only` → 然后实际运行一次 → 审查输出）再决定是否执行 `--all`

### `query --files <list> [--format ids|paths|json]`

反向查询：给定一组文件路径，输出覆盖它们的知识 ID。

```bash
# 单文件，默认 ids 格式
node packages/project-context/dist/cli.js query \
  --files packages/maker-core/src/agents/base-agent.ts

# 多文件 + JSON 格式
node packages/project-context/dist/cli.js query \
  --files packages/maker-core/src/agents/base-agent.ts,apps/desktop/src/main/index.ts \
  --format json

# 输出：
# {
#   "ids": ["apps--desktop", "packages--maker-core"],
#   "paths": [".../modules/apps--desktop.md", ".../modules/packages--maker-core.md"],
#   "stale": []
# }
```

**输出格式：**

| `--format` | 输出 |
|---|---|
| `ids`（默认） | 每行一个 ID |
| `paths` | 每行一个 .md 绝对路径 |
| `json` | 包含 `ids` / `paths` / `stale` 三个数组 |

主要供 agent harness 使用：将修改文件传入 → 获取 .md 路径 → 拼接到 system prompt。

---

## 产出物（`.cindy/project-knowledge/` 目录结构）

```
<repo-root>/.cindy/project-knowledge/
├── config.yaml          # 用户可编辑配置（init 时写入默认值）
├── manifest.yaml        # 索引（机器维护，可从 .md 重建）
├── .lock                # update 运行时的临时锁文件（自动清理）
├── modules/
│   ├── apps--desktop.md
│   ├── packages--maker-core.md
│   └── ...
└── concerns/            # 空目录，等待第二阶段自动填充
```

**单份模块 .md 文件示例：**

```markdown
---
id: packages--maker-core
type: module
covers:
  - packages/maker-core/**
depends_on: []
last_synced_commit: 4cc6490af1399d087977a3e62994264d6fe94fc1
last_synced_at: '2026-05-11T09:44:49.539Z'
stale: false
stale_reason: null
auto_update: true
schema_version: 1
---
# packages--maker-core

## 是什么
...

## 关键抽象 / 核心代码地标
...

## 模块边界
...

## 不要做的事
...

## 演进备忘
（仅追加，记录重大改动）
```

**Frontmatter 字段：**

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | string | 全局唯一，通常为路径转 kebab 格式 |
| `type` | `module` \| `concern` | 决定存放的子目录 |
| `covers` | glob[] | 反向查询代码 → 知识的关键 |
| `depends_on` | id[] | 注入时一跳扩展（“使用 A 必带 B”，第二阶段） |
| `last_synced_commit` | sha | diff 起点 |
| `last_synced_at` | ISO time | 用于调试 |
| `stale` | bool | 标记是否过期 |
| `stale_reason` | string \| null | 用于调试，记录触发 stale 的提交 |
| `auto_update` | bool（默认 true） | 设为 false 则 update 完全跳过此知识 |
| `schema_version` | int | 用于格式升级 |

---

## 配置（`.cindy/project-knowledge/config.yaml`）

`init` 首次运行时将写入默认配置，后续可手动编辑：

```yaml
# 自动发现模块的根路径（不写则按包管理器约定）
module_roots:
  - packages/*
  - apps/*

# 额外忽略规则（叠加在 .gitignore + 内置默认规则之上）
# 内置默认规则包括：node_modules, dist, build, *.lock, *.min.js, *.snap 等
ignore:
  - apps/some-vendored-thing

# 小 diff 阈值
small_diff_threshold:
  files: 5
  lines: 200

# 适配器
agent: claude-code              # MVP 仅支持 claude-code，未来支持 codex / custom
agent_options:
  timeout: 120                  # 秒，超时终止进程
  # model: opus                 # 可选，传给 claude --model
  # command: /path/to/claude    # 可选，覆盖默认 PATH 搜索
```

---

## 适配器（LLM 如何被调用）

MVP 内置一个适配器：`claude-code`。

**机制**：在子进程中 `spawn('claude', ['-p', '--output-format', 'text'])`，prompt 通过 stdin 传入，结果从 stdout 读取。

**前置条件**：
- 本机 PATH 中有 `claude` 可执行文件（[Claude Code CLI](https://docs.anthropic.com/claude-code)）
- 已登录（`claude` 自带的 OAuth 凭证）—— **无需 ANTHROPIC_API_KEY**

**失败模式**：
- `claude` 不在 PATH → 报错“`claude` CLI not found in PATH”
- 退出码非 0 → 报错并捕获 stderr
- 超时 → 终止子进程并报错
- 任何失败 → 该模块标记为 stale，update 继续处理后续 ID

后续可按 `src/adapters/types.ts` 中的接口添加 `codex` / `custom` 适配器。

---

## 常见接入工作流

### Cindy Desktop（本仓库默认 harness，开箱即用）

Cindy Desktop 已内置自动注入：每次创建新的 cc-agent / codex 会话时，主进程会探测当前工作目录下是否存在 `.cindy/project-knowledge/TOC.md`，存在则将该预渲染 TOC 包装为 `<project-context-toc>` wrapper 注入，不存在则静默跳过。

- **无需任何 UI 开关**：所有用户默认启用（不再走“实验功能”入口）
- **目录探测**：以当前工作目录为根，命中 `.cindy/project-knowledge/TOC.md` 才触发
- **per-session 快照**：是否注入会回写到 `sessions.used_project_context` 列；旧会话保持启动时刻状态，不受后续追溯影响
- **可视提示**：聊天页脚的 Brain 图标 + 工具提示“项目知识库已加载”（4 种语言已对齐）

注入逻辑实现：`apps/desktop/src/main/maker-ipc/projectContextInject.ts`。100KB 硬上限，超出则截断尾部并添加 `truncated="true"` 标记。

### 自动更新：本地 git post-commit 钩子（推荐主路径）

在仓库根目录的 `.git/hooks/post-commit`（手动创建文件，添加可执行权限）：

```bash
#!/bin/sh
# 异步、后台运行，不阻塞提交
(node /absolute/path/to/packages/project-context/dist/cli.js update > /tmp/pctx.log 2>&1) &
```

或团队场景使用 husky / lefthook 等工具管理。

### 让 agent 启动时获取知识：harness 集成

任何 agent harness（Claude Code、Codex CLI、Cindy、Cursor）在启动新会话前调用一次：

```bash
# 收集相关文件
files=$(git diff --name-only main...HEAD | tr '\n' ',')

# 查询相关 .md 路径
paths=$(node packages/project-context/dist/cli.js query --files "$files" --format paths)

# 拼接到 system prompt（具体拼接方式取决于 harness）
for p in $paths; do
  cat "$p"
done > /tmp/system-prompt-append.md
```

### 将 update 接入 Claude Code 的 Stop 钩子

在 Claude Code 的 hooks 配置中添加：

```jsonc
{
  "Stop": "node /absolute/path/to/packages/project-context/dist/cli.js update"
}
```

每次 agent 完成一轮即运行 update —— 适合 agent 频繁修改代码但不立即提交的场景。

---

## 如何手动控制某个模块

**冻结某个模块（禁止 update 修改）：**

编辑 `.cindy/project-knowledge/modules/<id>.md` 的 frontmatter，添加：

```yaml
auto_update: false
```

下次 update 遇到该模块将直接跳过（输出 `r <id> (auto_update=false)`）。

**清理孤立模块：**

```bash
# 默认仅报告
node packages/project-context/dist/cli.js init
# 实际删除
node packages/project-context/dist/cli.js init --prune
```

**强制重新生成某模块：**

```bash
# 单个
node packages/project-context/dist/cli.js refresh packages
```