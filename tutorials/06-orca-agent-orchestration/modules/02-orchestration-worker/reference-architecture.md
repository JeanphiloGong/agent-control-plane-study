# Orca Orchestration / Worker：参考架构

~~~text
TaskStore → Coordinator → DispatchStore
                 ↓              ↓
          PlacementPort     WorkerRuntimePort
                 ↓              ↓
          Workspace      Receipt / Observation
                 └────→ SettlementGate → TaskProjection
~~~

## 学习实现目录

~~~text
reference/orchestration/task.ts       # Task, Run and worker assignment
reference/orchestration/dispatch.ts   # Dispatch phase and attempt identity
reference/orchestration/receipt.ts    # stage/state projection
reference/orchestration/settlement.ts # report match and idempotency
reference/orchestration.test.ts       # V1–V3 boundary checks
~~~

| Orca 源码 | 参考组件 | 验收 |
| --- | --- | --- |
| worker-start-method.ts:startWorker | Coordinator.start | placement 和 owner 在 mutation 前确定 |
| worker-start-receipt.ts | Receipt | stage 与 state 分离 |
| worker-start-agent-view.ts | WorkerProjection | CLI 不隐藏 unknown |
| worker-report-settlement.ts | SettlementGate | report 只更新匹配的 Dispatch |
| orchestration-worker-settlement.ts | PostVerify | unknown 结果要求查询，不重复启动 |

V1 在 Worktree 的 Workspace 和 identity.key 上建立，V2/V3 继续复用同一 Task 和 Dispatch，不把 worker 过程重新建模成一个孤立命令。
