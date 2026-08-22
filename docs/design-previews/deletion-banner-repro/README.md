# 注销横幅被覆盖 Bug · 修复前复现 Demo（deletion-banner-repro）

> **本 Demo 是「修复前证据」**：复现 `feat/login-deletion-bubble` 修复的 P1 Bug——账号注销状态横幅被登录面板 100% 覆盖不可见（桌面端 + 移动端）。与其配对的修复后验收 Demo 位于 `../login-deletion-bubble/`。

## 事实钉在 Base Commit（不随分支漂移）

- 提取基线 = **本 PR 的 Base Commit `75baac1ee`**（origin/main 分叉点；该 Commit 的 main 源码即修复前状态）。
- `extract.mjs` 的源码读取**全部通过 `git show 75baac1ee:<仓库相对路径>`**（execFileSync，cwd=仓库根目录），**不读取工作区文件**——工作区已是修复后代码，读取则不再是「修复前」。
- 每个事实叶子的 `provenance.hash` 按固定字节计算；`provenance.source` 指向 `_pinned/`（提取时由 `git show` 落盘的源文件全集，先清空再写入，与提取严格一致）。**`_pinned/` 已加入 gitignore 不入仓**——extract 每次运行自动再生（确定性字节），仓库中不存放产品源文件副本（reviewer 不可见、grep 不命中）；skill 校验器按磁盘文件复核 hash，本地 extract 运行通过即可验证。
- 为何钉 SHA 而非工作区：本 Demo 断言的是**修复前**的结构事实（如 `makeStyles.deletionStatus` 无底色、横幅在 `{stateContent}` 之前的渲染顺序），修复后这些结构已不存在；钉 SHA 后漂移检查恒定有意义（语义 = 提取器或固定基线被改动），读取工作区则必然失效。

## 如何重跑

```bash
# 漂移检查（extract 现跑 ≡ truth.json + provenance 校验）
node ~/.claude/skills/qa-hifi-demo/scripts/truth.mjs --demo docs/design-previews/deletion-banner-repro --check
# 门 A-D/F
node ~/.claude/skills/qa-hifi-demo/scripts/verify.mjs --demo docs/design-previews/deletion-banner-repro
# 自定义重叠门（重叠率 100% / elementFromPoint 命中面板 / 意图帧自证）
# 本仓库 node_modules 无 playwright 时，用 QA_HIFI_MODULE_ROOT 指向已安装的目录
QA_HIFI_MODULE_ROOT=<已安装 playwright 的目录> node docs/design-previews/deletion-banner-repro/overlap-gate.mjs
```

当前基线结果（2026-07-26，@75baac1ee）：verify 门 A/B（24/24）/C/D（496/496）全绿、门 F 未配置、overlap-gate 12/12——**Bug 在 Base Commit 上必然存在**，这正是「修复前证据」要证明的事。

## 目录

- `spec.json` / `extract.mjs` / `truth.json` / `index.html` / `report.json` — qa-hifi-demo 标准五件套
- `overlap-gate.mjs` — 自定义重叠门脚本
- `_pinned/` — `git show 75baac1ee` 落盘的 provenance 源文件（**已 gitignore，extract 每次运行自动再生，不入仓**）
- `evidence/` — overlap-gate.json + 目检截图