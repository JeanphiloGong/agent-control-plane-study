# Scheduler / Lease：参考架构

```text
Manager.RunOnce → dbNow → runJob
                          ├─ markStaleAsFailed → FAILED
                          └─ plansForTick → processPlan → tryClaim
                                              └─ runClaimed(ID, lease token)
                                                   ├─ handler + heartbeat
                                                   └─ finishSuccess / finishFailure
```

`manager.go` 负责计划与执行编排。`db_ops.go` 通过 PostgreSQL 的唯一键和带 token 的条件更新维护 `sys_cron_executions`。教程的 `Store` 用单进程临界区呈现相同的领取与写入规则，实际行更新由 `stale_steal_test.go` 等源码测试验证。

V1 验收竞争领取和终态写入。V2 验收心跳续期。V3 验收停止心跳后的接管、失败重试和对旧 token 的拒绝。
