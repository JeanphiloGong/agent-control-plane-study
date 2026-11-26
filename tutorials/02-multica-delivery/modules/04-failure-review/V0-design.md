# Multica Failure / Review V0

[模块路线图](plan.md) · [参考架构](reference-architecture.md)

## 版本与范围

- 源码：server/pkg/taskfailure/*、server/internal/service/task.go、issue_wakeup*.go、packages/core/issues/*
- 目标：把 Run 结果分类为失败、可重试、需等待或需审核，并回到 Issue projection。
- 排除：完整 Squad 编排、所有外部渠道。

## 真实流程

~~~text
Run terminal result
  → TaskService.FailTask / completed task
  → taskfailure.Classify
  → retryableReasons / CreateRetryTask
  → IssueWakeupService.Tick
  → task / Issue projection
~~~

FailTask 负责任务终态、失败原因、retry child 和聊天失败结果。它不直接把 Issue 改成 shipped。IssueWakeupService 负责规则条件、到期时间、权限和 receipt 去重。客户端投影读取服务端结果，不重新判断 retryability。

## 源码对应

| 路径 | 符号/责任 |
| --- | --- |
| server/pkg/taskfailure/failure.go | 稳定 Reason 值、IsAgentError、AllReasons |
| server/pkg/taskfailure/classify.go | Classify、NormalizeDaemonReason、ResumeUnsafeFailure |
| server/internal/service/task.go | FailTask、retryableReasons、retryAttemptCeiling、retryDelayForAttempt、MaybeRetryFailedTask |
| server/internal/service/issue_wakeup.go | IssueWakeupService.Tick、dispatchFromTick |
| server/internal/service/issue_wakeup_condition.go | 条件评估与 condition.met 事件 |
| packages/core/issues/run-steering.ts | Run 状态的客户端行为投影 |
| packages/core/issues/config/status.ts | in_review、blocked 等状态词 |

## 前置、产出与交接

- 前置：03 Scheduler 的 terminal result 和 lease outcome。
- V1 产出：稳定失败分类与 Review Gate 决策。
- V2 产出：有限 retry child，保留 Issue/workflow handoff。
- V3 产出：Issue Wakeup 条件去重和客户端状态投影。
- 下游：M1 继续处理完整 Review 操作和交付审核。
- 冻结条件：Retryable、ReviewRequired 和 Issue status 是三个独立事实；成功 Run 不能自动成为 shipped。

## V1 检查

provider network 进入可重试决策，权限错误进入 blocked/review，业务失败不自动重试，成功进入 in_review。

实现正文从 v1/failure-review.md 开始。
