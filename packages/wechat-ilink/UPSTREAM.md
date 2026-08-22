# 腾讯 iLink 协议溯源

协议行为及部分纯工具逻辑独立改编自：

- 仓库：<https://github.com/Tencent/openclaw-weixin>
- 标签：`v2.4.6`
- 提交：`cef0bfc390393f716903e16d50408118047f87e0`
- 获取时间：2026-07-27
- 许可证：MIT；参见 `LICENSE.tencent-openclaw-weixin`

衍生或行为适配区域：

- `src/types.ts`：`src/api/types.ts`
- `src/apiClient.ts`：`src/api/api.ts`
- `src/auth.ts`：`src/auth/login-qr.ts`
- `src/mediaCrypto.ts`：`src/cdn/aes-ecb.ts`
- `src/markdown.ts`：`src/messaging/markdown-filter.ts`

Cindy 不复制 OpenClaw 插件运行时、文件系统持久化、账户存储、日志记录或主机集成。存储、取消操作、凭据及网络执行均明确由主机负责。