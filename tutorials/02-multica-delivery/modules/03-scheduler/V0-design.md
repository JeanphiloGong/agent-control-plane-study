# Multica Scheduler / Lease V0

## 版本与范围

- **源码**：`server/internal/scheduler/db_ops.go`、`manager.go`、`stale_steal_test.go`
- **目标**：解释 claim、lease token、数据库时间、heartbeat、stale steal 和 terminal update 的围栏。
- **排除**：多区域队列、完整 worker autoscaling。

## 真实流程

```text
Manager.RunOnce → tryClaim(db clock)
  → runClaimed(lease token)
  → runHeartbeats / heartbeat
  → finishSuccess or finishFailure(token)
```

如果 heartbeat 停止，`stale_after` 到期，另一个 runner 可以接管并取得新 token。旧 runner 后续调用 `finishSuccess` 必须返回 `ErrLeaseLost`，不能覆盖胜者。

## 源码对应

| 路径 | 符号/责任 |
| --- | --- |
| `server/internal/scheduler/db_ops.go` | `dbNow`、`tryClaim`、`heartbeat`、`finishSuccess`、`finishFailure`、`ErrLeaseLost` |
| `server/internal/scheduler/manager.go` | `Manager.RunOnce`、`runClaimed`、`runHeartbeats`、错误分类 |
| `server/internal/scheduler/stale_steal_test.go` | `TestStaleStealTerminalUpdateIgnored`，验证旧 lease 终态写入被拒绝 |

## 局部版本线

- V1：单 worker claim 和 terminal update。
- V2：lease token 与 heartbeat。
- V3：stale steal、失败重试和并发竞争。

## V1 检查

两个 worker 竞争同一 job，只能一个获得 claim。让 owner 停止 heartbeat，第二个 worker 接管，再用旧 token finish，断言 `ErrLeaseLost`。

契约见 `contracts/`，参考架构见 `reference-architecture.md`。
