# 04 Failure / Review：从 Run 结果到 Retry、Wakeup 与 Issue 状态

[Multica 总览](../../V0-overview.md) · [V0 设计](V0-design.md) · [参考架构](reference-architecture.md)

03 Scheduler / Lease 负责一个系统任务是否被当前 runner 持有。本模块处理 Agent Run 结束后的业务事实：错误是什么、是否允许有限重试、是否需要 Review，以及何时由 Issue Wakeup 再次进入执行。

| 版本 | 前置 | 实现正文 | 验收 |
| --- | --- | --- | --- |
| v1 | V0 契约 | [失败分类与 Review Gate](v1/failure-review.md) | 网络、权限、业务失败和成功结果进入不同决策 |
| v2 | v1 Decision | [有限重试与 workflow handoff](v2/retry-policy.md) | 次数、延迟、预算和 lineage 有边界 |
| v3 | v2 RetryPlan | [Wakeup 与状态投影](v3/wakeup-projection.md) | 条件不满足不唤醒，并发 Tick 只 dispatch 一次，成功保持 in_review |

## 真实输入与输出

~~~text
TaskService.FailTask / completed task
  → taskfailure.Classify
  → retryableReasons + retryAttemptCeiling
  → CreateRetryTask(fire_at, max_attempts)
  → IssueWakeupService.Tick
  → task enqueue / Issue projection
~~~

FailTask 在同一事务中结算失败 task 和 retry child。IssueWakeupService.Tick 另有自己的 wakeup 行锁和 receipt 去重。客户端只展示服务端的 task/Issue 状态，不能自行推导 retryability 或 Review approval。

## 交接边界

Scheduler lease 丢失由 03 负责。这里消费 terminal result 和 failure reason。Issue 的最终 shipped 仍然需要 Review 或工作流动作，本模块不会自动提升成功 Run。

## 验证方式

参考项目从 v1/reference/ 的 Go module 继续。每版运行 go test ./...，V3 追加 go test -race ./...。真实 taskfailure 单元测试可在 sources/multica/server 内运行，数据库依赖的 retry/wakeup 测试需要独立 fixture。
