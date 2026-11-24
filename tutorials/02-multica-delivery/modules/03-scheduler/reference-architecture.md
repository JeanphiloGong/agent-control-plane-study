# Scheduler / Lease：参考架构

```text
SchedulerManager → ExecutionStore.try_claim
       ↓ lease token
Runner ──heartbeat──> ExecutionStore
Runner ──finish(token)──> fenced terminal update
Stale scanner ──database clock──> steal with new token
```

参考实现使用 PostgreSQL 表或等价的事务 store，不能用进程内布尔变量代替 lease。源码 `db_ops.go` 是存储边界，`manager.go` 是调度编排，`stale_steal_test.go` 是失败契约。

V1 冻结检查是竞争 claim、停止 heartbeat 后接管、旧 token finish 被拒绝。
