# Orca 03 Session / Recovery：V0 设计

[Orca 总览](../../V0-overview.md) · [模块计划](plan.md) · [参考架构](reference-architecture.md)

## 版本身份

- **模块**：orca-session-recovery
- **基线**：sources/orca @ 27300a14e0170258814b67b2f4159446c637f551
- **模式**：backend / design
- **目标**：把 worker session 的运行、停止、lost、retry 和 preserve 分成可以证明的事实。
- **前置**：02 Orchestration / Worker V3 提供 Dispatch、session identity 和 settlement。
- **V1 交接**：对一个 Dispatch 记录来自所属 host 的 liveness 观察。

本版不把 silence 当成 exit，也不允许 recovery 直接删除 workspace。停止、重试和 cleanup
必须分别使用自己的证据边界。

## 真实场景

Worker 启动后，Coordinator 可能暂时读不到它。远程断线、启动超时、进程退出和 worker
主动报告失败都需要不同处理。只有确切的 owned identity、host 观测和 settled predecessor
才能允许 retry。未知状态下默认保留工作区。

~~~text
worker dispatch
  → session/terminal observation
  → live / exited / unknown
  → stop or inspect
  → same-Task retry after confirmed cause
  → preserve workspace or send cleanup evidence
~~~

## 领域对象和真实入口

| 对象 | 真实源码 | 责任 |
| --- | --- | --- |
| Session observation | src/main/runtime/orchestration/workers/lifecycle/session-worker-liveness.ts | 记录 worker/session 的可验证存活信息 |
| Recovery command | src/cli/orchestration/recovery/orchestration-recovery-command.ts | 读取 Dispatch 和 Task，再决定恢复动作 |
| Stop receipt | src/main/runtime/rpc/methods/orchestration/worker/stop/worker-stop-remote-receipt.ts | 表达远程停止成功、未知和能力差异 |
| Worker control | src/main/runtime/rpc/methods/orchestration/worker/worker-control.ts | show、read、abandon 等带身份约束的操作 |
| Retry contract | skill-guides/orchestration/references/recovery-and-cleanup.md | 要求同一 Task、最新 predecessor、原因变化和 circuit breaker |
| Cleanup evidence | src/shared/workspace-cleanup/workspace-cleanup.ts | 将 worker running、terminal unknown 等事实变成 blocker |

## 业务不变量

1. unknown liveness 不是 exited，不能自动 stop、retry、release 或 abandon。
2. stop 必须按确切 Dispatch ID 和 owned resource 执行，stop_unknown 需要检查而不是盲目重复。
3. retry 使用同一个 Task 和最新失败 attempt，不能以新 Task 绕过失败链。
4. 同一 failure cause 不改变时不能自动 retry，连续失败达到 breaker 后停止自动重试。
5. worker 的失败或退出不能证明 worktree 可删除；未完成工作默认保留。
6. release 只处理已证明可重用的 owned terminal，不能关闭用户接管或身份未知的进程。

## V1–V3 交接

- **V1 Liveness**：把 host 观测写入 Dispatch，保留 unknown 与 exited 的区别。
- **V2 Retry**：确认停止和原因变化后，在同一 Task 创建新 Dispatch，旧 attempt 留在历史中。
- **V3 Preserve**：将所有相关 Dispatch 的存活证据交给 cleanup，未知或活动状态强制 keep。

## V0 验收

- 为什么启动超时不能直接当作进程退出？
- retry 为什么必须指向同一个 Task 的最新 predecessor？
- worker 已 stopped 但目录有 uncommitted files 时，哪个模块决定保留？
- coordinator restart 后，如何从 Dispatch ID 恢复而不是创建第二个 worker？

真实检查入口包括 src/main/runtime/orchestration/workers/lifecycle/session-worker-liveness.ts、src/main/runtime/rpc/methods/orchestration/worker/stop/worker-stop-remote-receipt.ts 和 skill-guides/orchestration/references/recovery-and-cleanup.md。

[领域流](contracts/domain-flow.md)、[接口与存储](contracts/interface-or-storage.md)、[状态与错误](contracts/state-and-errors.md) 和[参考架构](reference-architecture.md)定义后续实现边界。
