# Scheduler / Lease：领域流

```text
ready job → tryClaim → running(lease token, heartbeat_at, stale_after)
  → heartbeat repeatedly
  → done/failed only with current token
  → stale_after passed → stale steal with new token
```

Lease 是运行权限的时间边界，heartbeat 是持有者续期事实，terminal update 是受 token 围栏保护的写入。
