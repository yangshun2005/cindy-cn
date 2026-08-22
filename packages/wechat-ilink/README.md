# @cindy/wechat-ilink

与宿主无关的腾讯 iLink 协议客户端，用于 Cindy 的个人微信 IM 连接器。

该包负责线路编码、二维码授权、长轮询、文本发送、输入状态、消息解码、文本过滤/分块以及媒体加密原语。它刻意不负责：

- 凭据或游标持久化；
- 文件系统路径或临时文件；
- Electron、Renderer、SQLite 或 Maker 集成；
- 日志记录或遥测；
- 重试队列和任务执行。

宿主必须为每次操作注入 `fetch`、一个 `AbortSignal`，以及任何验证码用户体验。授权挑战仅暴露用于显示的二维码内容；轮询令牌和之前的机器人令牌保留在包内存中，并在授权完成后清除。

```ts
const transport = new TencentIlinkTransport({
  baseUrl: "https://ilinkai.weixin.qq.com",
  token,
  botAgent: `Cindy/${version}`,
  clientVersion: version,
  fetch,
});
```

仅接受 `weixin.qq.com` 下无凭据的 HTTPS 来源。错误以 `WechatIlinkError` 形式跨越包边界，带有稳定的错误码，且绝不包含响应体、授权头、二维码轮询令牌或机器人令牌。

上游协议来源和许可证详情记录在 [`UPSTREAM.md`](UPSTREAM.md) 中。