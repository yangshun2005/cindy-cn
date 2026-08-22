# @cindy/heartbeat-client

Cindy 客户端在线心跳模块。

- 零运行时依赖，仅用全局 `fetch` + `AbortSignal.timeout`
- 与 Electron 无关（浏览器 / Node / Electron 主进程都能运行）
- Host 通过依赖注入提供 `getUid()` / `getPlatform()` / `getVersion()` / `logger`
- 启动时立即发送一次，后续按 `intervalMs` 周期进行 tick
- 静默失败：任何网络/超时/非 2xx 仅发出警告，绝不抛出异常
- 当 `getUid()` 返回 null 时自动跳过本次（适用于登录态切换间隙）

## 使用方法

```ts
import { createHeartbeatClient } from '@cindy/heartbeat-client';

const handle = createHeartbeatClient({
  endpoint: 'https://heartbeat.example.com',
  intervalMs: 60_000,
  timeoutMs: 5_000,
  host: {
    getUid: () => currentUser?.id ?? deviceId,
    getPlatform: () => process.platform,
    getVersion: () => app.getVersion(),
    logger: myLogger,
  },
});

// 退出时：
handle.stop();
```

## 服务端约定

配套服务端：`apps/heartbeat-server`。仅有一个 endpoint：

- `POST {endpoint}/heartbeat` 请求体 `{ uid, platform?, version? }` → `{ ok: true }`

无需鉴权。