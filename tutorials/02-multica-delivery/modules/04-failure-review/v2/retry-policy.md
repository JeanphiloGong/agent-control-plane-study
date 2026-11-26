# Multica Failure / Review v2：创建有边界的 Retry child

[上一版 v1](../v1/failure-review.md) · [本版计划](plan.md) · [模块路线图](../plan.md) · [下一版 v3](../v3/plan.md)

V1 只回答“能不能重试”。实际系统还必须知道第几次尝试、什么时候可以再次领取，以及 retry 是否仍属于同一个 Issue workflow。

## 1. 在参考模型上加入 Retry 状态

V2 继续使用 v1/reference/review/model.go 中的 Run、Decision、RetryPlan 和 RetryState。这些类型已经在 V1 的完整模型文件中出现，V2 只开始使用 RetryPlan。

创建 v1/reference/review/retry.go：

~~~go
package review

import "time"

func Retry(parent Run, now time.Time) (RetryPlan, bool) {
	decision := Decide(parent)
	if !decision.Retryable || parent.Attempt >= retryCeiling(parent) {
		return RetryPlan{}, false
	}
	next := parent.Attempt + 1
	delay := time.Duration(0)
	state := RetryQueued
	if decision.Reason == ReasonProviderNetwork && parent.Attempt >= 2 {
		delay = 5 * time.Second
		state = RetryDeferred
	}
	child := parent
	child.ID = parent.ID + "/attempt-" + string(rune('0'+next))
	child.Error = ""
	child.Attempt = next
	child.MaxAttempts = retryCeiling(parent)
	fireAt := time.Time{}
	if delay > 0 {
		fireAt = now.Add(delay)
		child.Status = RunDeferred
	} else {
		child.Status = RunQueued
	}
	return RetryPlan{Child: child, State: state, FireAt: fireAt}, true
}

func retryCeiling(parent Run) int {
	if parent.MaxAttempts <= 1 {
		return parent.MaxAttempts
	}
	if Classify(parent.Error) == ReasonProviderNetwork && parent.MaxAttempts < 3 {
		return 3
	}
	return parent.MaxAttempts
}
~~~

Retry 只对 V1 分类为可重试且还有预算的 Run 创建 child。provider network 的通用预算为三次，原任务的 MaxAttempts=1 仍然明确关闭自动重试。第一次 child 的状态是 queued，最终一次是 deferred，并增加 5 秒 fire_at 延迟。child 保留 IssueID、WorkflowStep 和 Handoff，所以重试不会变成一个脱离原交接上下文的新任务。

真实源码的对应逻辑位于 server/internal/service/task.go。retryableReasons、retryAttemptCeiling 和 retryDelayForAttempt 共同决定原因白名单、有效次数上限和 fire_at。FailTask 在同一个事务中把父任务改为 failed 并调用 CreateRetryTask。本教程的 Retry 返回值对应这个事务前的决定，不伪装成已经写入 PostgreSQL。

## 2. 验收次数、时间和 workflow lineage

创建 v1/reference/review/retry_test.go：

~~~go
package review

import (
	"testing"
	"time"
)

func TestRetryPreservesIssueWorkflowAndUsesBoundedSchedule(t *testing.T) {
	now := time.Unix(100, 0).UTC()
	parent := Run{ID: "run-1", IssueID: "issue-1", WorkflowStep: "delivery", Handoff: "keep brief", Status: RunFailed, Error: "connection closed", Attempt: 1, MaxAttempts: 2}
	first, ok := Retry(parent, now)
	if !ok || first.State != RetryQueued || first.Child.Attempt != 2 || first.Child.MaxAttempts != 3 {
		t.Fatalf("first retry = %+v, ok=%v", first, ok)
	}
	if first.Child.IssueID != parent.IssueID || first.Child.WorkflowStep != parent.WorkflowStep || first.Child.Handoff != parent.Handoff {
		t.Fatalf("retry lost workflow identity: %+v", first.Child)
	}
	first.Child.Error = "connection closed again"
	second, ok := Retry(first.Child, now)
	if !ok || second.State != RetryDeferred || !second.FireAt.Equal(now.Add(5*time.Second)) {
		t.Fatalf("second retry = %+v, ok=%v", second, ok)
	}
	exhausted := second.Child
	exhausted.Error = "connection closed"
	if _, ok := Retry(exhausted, now); ok {
		t.Fatal("retry exceeded the effective attempt ceiling")
	}
}

func TestRetryDoesNotCreateForAuthFailure(t *testing.T) {
	parent := Run{ID: "run-2", Status: RunFailed, Error: "401 invalid api key", Attempt: 1, MaxAttempts: 3}
	if _, ok := Retry(parent, time.Unix(100, 0)); ok {
		t.Fatal("auth failure was retried")
	}
}
~~~

运行：

~~~bash
cd tutorials/02-multica-delivery/modules/04-failure-review/v1/reference
go test ./...
~~~

预期四个测试通过。它们证明第一次 retry 可立即领取，第二次 retry 等待明确的 fire_at，次数耗尽后停止，权限失败不创建 child，并且 Issue/workflow/handoff 字段完整保留。

真实数据库边界可对照 server/internal/service/retry_deferred_test.go 中的 TestCreateRetryTaskFireAtControlsDeferral、TestFailTaskProviderNetworkBudget 和 TestRetryKeepsWorkflowHandoffBrief。运行这些测试需要 Multica 的测试数据库和完整迁移，当前教程回放不修改该数据库。

## 检查点与下一版

V2 只生成 retry 计划。下一版把到期条件接到 Issue Wakeup，并把 Run 决策投影为客户端可显示的 Issue 状态。
