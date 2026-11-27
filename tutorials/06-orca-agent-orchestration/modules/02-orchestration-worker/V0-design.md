# Orca 02 Orchestration / Worker：V0 设计

[Orca 总览](../../V0-overview.md) · [模块计划](plan.md) · [参考架构](reference-architecture.md)

## 版本身份

- **模块**：orca-orchestration-worker
- **基线**：sources/orca @ 27300a14e0170258814b67b2f4159446c637f551
- **模式**：backend / design
- **目标**：把一个 Coordinator Task 转成可追踪的 Dispatch，并区分 worktree created、session ready、turn started、worker report 和 Task settlement。
- **前置**：01 Worktree / Workspace V1 提供稳定 workspace identity。
- **V1 交接**：创建一次 Dispatch，并把它绑定到 Task、Run、Worker 和 Workspace。

本版只冻结状态边界。它不启动真实 Provider，也不把 CLI receipt 当成 Task 的最终事实。

## 真实场景

Coordinator 接收一个带执行说明的 Task，选择 worker、Run、host 和 worktree，然后启动
worker。创建 workspace、session ready 和 turn started 是三个不同事实。worker 最后报告
head、检查结果和 outcome，Coordinator 仍要根据 Dispatch ID、session ID 和 Task 关系
确认结算。

~~~text
coordinator task
  → placement and worktree
  → Dispatch attempt
  → session ready
  → turn started
  → worker report
  → dispatch settlement
  → Task outcome
~~~

## 领域对象和真实入口

| 对象 | 真实源码 | 责任 |
| --- | --- | --- |
| Task | src/cli/specs/orchestration.ts、src/main/runtime/orchestration/db/tasks/* | 保存目标说明、Run 所属和 Task status |
| Dispatch | src/main/runtime/orchestration/db/worker-dispatch/* | 保存一次 worker attempt、ID、状态和 predecessor |
| Worker receipt | src/main/runtime/rpc/methods/orchestration/worker/start/worker-start-receipt.ts | 把启动阶段、状态、错误和 next action 返回给调用方 |
| Worker start | src/main/runtime/rpc/methods/orchestration/worker/worker-start-method.ts | 解析 Task、placement、agent 和 retry，创建或复用 worker |
| CLI view | src/cli/orchestration/workers/worker-start-agent-view.ts | 将 runtime receipt 投影为用户可读的 worker 状态 |
| Settlement | src/main/runtime/orchestration/db/dispatch-context/worker-report-settlement.ts、src/cli/orchestration/workers/orchestration-worker-settlement.ts | 用 Dispatch、Run、Task 和 worker report 确认终态 |

## 业务不变量

1. 同一个 Task 的不同 worker 可以有不同 Dispatch，但一次 worker 的当前 attempt 必须可唯一定位。
2. workspace-created 不等于 session-ready，session-ready 也不等于 turn-started。
3. Receipt 是调用结果投影，Dispatch 和 Task 是持久事实，不能以 receipt 单独结算。
4. worker report 必须匹配当前 Dispatch ID、session ID、Run 和 Task，迟到或重复 report 不能覆盖新 attempt。
5. 同一份成功 report 重放是幂等的，内容不同的第二份 report 必须拒绝。
6. Task settlement 保留旧 Dispatch，当前 outcome 只由每个 worker 的最新 attempt 汇总。

## 失败边界

- worktree 创建失败要产生明确的 failed stage，不能返回一个可继续使用的 live worker。
- session ready 未确认时，turn-started 请求拒绝。
- runtime 接受 mutation 但 post-verification 无法确认时，结果是 operation unknown，调用方不能新建 Task。
- worker 自己报告失败和 worker 进程 lost 是不同事实，后者进入 recovery 模块。
- 单个 worker 失败不应伪造其他 worker 已完成，也不能直接把 Review 当成 merge。

## V1–V3 交接

- **V1 Dispatch**：Task、Run、Worker、Workspace 和一次 Dispatch ID 建立绑定。
- **V2 Receipt**：用受控观测推进启动阶段，区分 session ready 与 turn started。
- **V3 Settlement**：校验 worker report，保持幂等并汇总多个 worker 的当前 outcome。

## V0 验收

必须能解释以下问题：

- 为什么 receipt 的 state: ready 不能证明 turn 已经开始？
- 哪些字段能阻止旧 session 的 report 写入新 Dispatch？
- 两个 worker 都完成时，Task 如何从 running 变成 succeeded？
- 一个 worker failed 时，为什么另一个 worker 的成功事实仍需保留？

真实测试入口包括 src/main/runtime/rpc/methods/orchestration/worker/start/worker-start-receipt.test.ts、worker-start-selector-refusal.test.ts 和 src/cli/orchestration/workers/orchestration-worker-settlement.ts。

[领域流](contracts/domain-flow.md)、[接口与存储](contracts/interface-or-storage.md)、[状态与错误](contracts/state-and-errors.md) 和[参考架构](reference-architecture.md)是 V1 的设计交接。
