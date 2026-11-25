# Multica Scheduler / Lease V0

[模块路线图](plan.md)

## 版本与范围

- **源码**：`server/internal/scheduler/db_ops.go`、`manager.go`、`stale_steal_test.go`
- **目标**：解释 claim、lease token、数据库时间、heartbeat、stale steal 和 terminal update 的围栏。
- **排除**：多区域队列、完整 worker autoscaling。

## 真实流程

```text
Manager.RunOnce → dbNow → runJob → plansForTick
  → processPlan → tryClaim(job, scope, planTime)
  → runClaimed(executionID, leaseToken)
       ├─ handler + runHeartbeats
       └─ finishSuccess / finishFailure
```

`cmd/server/main.go` 在启动时注册系统任务。`Manager.RunOnce` 使用数据库时间为每个 job 计算到期计划。若 `RUNNING` 行已超过 `stale_after`，且该 job 允许接管并有剩余尝试，另一个 runner 可以在同一行上领取新 token。旧 runner 后续提交时会得到 `ErrLeaseLost`。

## 源码对应

| 路径 | 符号/责任 |
| --- | --- |
| `server/internal/scheduler/db_ops.go` | `dbNow`、`tryClaim`、`heartbeat`、`finishSuccess`、`finishFailure`、`ErrLeaseLost` |
| `server/internal/scheduler/manager.go` | `Manager.RunOnce`、`runClaimed`、`runHeartbeats`、错误分类 |
| `server/internal/scheduler/stale_steal_test.go` | `TestStaleStealTerminalUpdateIgnored`，验证旧 lease 终态写入被拒绝 |

## 前置、产出与交接

- 学习顺序接在 Issue Invocation 与 Runtime Access 之后。代码从已算好的系统任务计划开始。 [Issue Invocation](../01-issue-invocation/V0-design.md) · [Runtime Access](../02-runtime-access/V0-design.md)
- V1 先实现竞争领取和受 token 保护的终态写入。V2 增加心跳，V3 增加重新领取。
- 后续模块继续讲 Agent Run 的失败分类、唤醒和审核。 [Failure / Review](../04-failure-review/V0-design.md)
- 同一计划同时只有一个当前持有者。失去 lease 后，旧 token 的终态更新影响零行。

## 局部版本线

- V1 实现竞争领取、plan 唯一键和终态更新。
- V2 实现带 token 的心跳续期。
- V3 实现过期接管和失败重试。

## V1 检查

两个 worker 竞争同一计划，只有一个获得 claim。用错误 token 尝试终态写入，断言 `ErrLeaseLost`。让心跳过期后再接管的场景在 V3 验收。

[领域流](contracts/domain-flow.md) · [存储契约](contracts/interface-or-storage.md) · [状态与错误](contracts/state-and-errors.md) · [参考架构](reference-architecture.md)
