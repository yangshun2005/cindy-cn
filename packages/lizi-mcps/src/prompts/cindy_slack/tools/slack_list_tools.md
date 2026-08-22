好的，这是将 `packages/lizi-mcps/src/prompts/cindy_slack/tools/slack_list_tools.md` 文件内容翻译为简体中文的结果：

---

列出 Slack 官方 MCP 在当前授权下可用的工具清单（名称 / 说明 / 参数 JSON Schema）。工具面由 Slack 根据授权 scope 动态决定：搜索消息 / 读取频道历史 / 查询用户 / 发送消息 / 添加 reaction 等。在首次操作 Slack 前先调用此工具，再使用 `slack_call_tool` 调用具体工具；清单在会话内基本不变，不要反复拉取。当设备绑定了多个 Slack workspace 时，传入 `team_id` 以指明按哪个 workspace 的授权获取清单（列表可用 `slack_status` 查询）。