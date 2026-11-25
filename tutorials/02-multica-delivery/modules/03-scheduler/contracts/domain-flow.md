# Scheduler / Lease：领域流

```text
due plan → tryClaim → RUNNING(lease token, heartbeat_at, stale_after)
  ├─ heartbeat → renew stale_after
  ├─ finish with current token → SUCCESS / FAILED
  ├─ stale + reentry allowed + attempts remain → RUNNING with new token
  └─ stale sweep → FAILED
FAILED + retry due + attempts remain → RUNNING with new token
```

`stale_after` 提供接管时间边界。当前 token 决定哪次尝试可以续期或提交终态。
