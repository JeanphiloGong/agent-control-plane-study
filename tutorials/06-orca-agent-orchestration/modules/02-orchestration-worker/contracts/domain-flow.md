# Orca Orchestration / Worker：领域流契约

~~~text
Task + Run
  → select workspace and worker
  → create Dispatch attempt
  → start worktree/session/turn
  → emit receipt and observation
  → accept worker report
  → settle Dispatch
  → aggregate current worker outcomes into Task
~~~

| 对象 | 所有者 | 不拥有 |
| --- | --- | --- |
| Task | Coordinator/Task store | Provider terminal 和 Git diff |
| Dispatch | orchestration store | 另一个 attempt 的 report |
| Receipt | runtime mutation response | 最终 Task truth |
| Worker report | worker session/settlement boundary | Review approval |
| Task outcome | coordinator aggregation | filesystem cleanup |

Dispatch 的生命周期事实应按 worktree-created → session-ready → turn-started → settled 观察，而不是用一个布尔字段覆盖所有阶段。
