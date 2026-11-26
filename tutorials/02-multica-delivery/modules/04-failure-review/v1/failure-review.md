# Multica Failure / Review v1：分类一次 Run，并把 Review 与 Retry 分开

[模块路线图](../plan.md) · [V0 设计](../V0-design.md) · [本版计划](plan.md) · [下一版 v2](../v2/plan.md)

## 版本定位

一次 Agent Run 结束时，status=failed 只说明执行没有完成。它还需要一个稳定的失败原因，才能决定是否可以自动重试。status=completed 也只说明 Agent 运行完成，Issue 是否可以交付仍由 Review 决定。

本版做一个可执行的决策边界。provider 网络中断进入有限 Retry，权限错误和业务失败进入 Review，成功结果进入 in_review。本版不创建数据库任务，也不调用 Issue API。参考项目用纯 Go 值对象复现这段决策，真实写入仍由 Multica 的 TaskService.FailTask 和 Issue 状态命令负责。

## 1. 建立参考项目和稳定模型

在学习仓库根目录执行：

~~~bash
mkdir -p tutorials/02-multica-delivery/modules/04-failure-review/v1/reference/review
cd tutorials/02-multica-delivery/modules/04-failure-review/v1/reference
go mod init example.com/multica-failure-review
~~~

创建 v1/reference/review/model.go：

~~~go
package review

import "time"

type RunStatus string

const (
	RunQueued    RunStatus = "queued"
	RunDeferred  RunStatus = "deferred"
	RunSucceeded RunStatus = "succeeded"
	RunFailed    RunStatus = "failed"
)

type FailureReason string

const (
	ReasonProviderNetwork FailureReason = "provider_network"
	ReasonProviderAuth    FailureReason = "provider_auth"
	ReasonBusiness        FailureReason = "business_failure"
	ReasonUnknown         FailureReason = "unknown"
)

type Run struct {
	ID           string
	IssueID      string
	WorkflowStep string
	Handoff      string
	Status       RunStatus
	Error        string
	Attempt      int
	MaxAttempts  int
}

type Decision struct {
	Reason         FailureReason
	Retryable      bool
	ReviewRequired bool
	IssueStatus    string
	UserMessage    string
}

type RetryState string

const (
	RetryQueued   RetryState = "queued"
	RetryDeferred RetryState = "deferred"
)

type RetryPlan struct {
	Child  Run
	State  RetryState
	FireAt time.Time
}

type WakeupRule struct {
	ID            string
	IssueID       string
	Instruction   string
	ExpectedState string
	Enabled       bool
	NextFireAt    time.Time
	LastDispatch  string
}

type IssueProjection struct {
	IssueID      string
	Status       string
	RunID        string
	Failure      FailureReason
	ReviewNeeded bool
}
~~~

Run 保存 Issue 与 workflow handoff 的身份，Decision 是本轮策略的结果。Retryable 和 ReviewRequired 是两个独立布尔值。成功可以要求 Review，失败也可以允许 Retry，不能用一个 status 字段代替两种判断。

真实源码中，taskfailure.Reason 的字符串会写入 agent_task_queue.failure_reason。ReasonAgentProviderNetwork 是 agent-side taxonomy 中唯一明确加入 TaskService 自动重试白名单的 provider 错误。FailTask 仍然只负责写 task 终态和 retry child，不直接把 Issue 写成已交付。

## 2. 实现分类和 Review Gate

创建 v1/reference/review/failure.go：

~~~go
package review

import (
	"strconv"
	"strings"
)

func Classify(raw string) FailureReason {
	lower := strings.ToLower(strings.TrimSpace(raw))
	if lower == "" {
		return ReasonUnknown
	}
	if strings.Contains(lower, "network") ||
		strings.Contains(lower, "connection closed") ||
		strings.Contains(lower, "dial tcp") ||
		strings.Contains(lower, "connection timeout") {
		return ReasonProviderNetwork
	}
	for _, code := range []string{"401", "403"} {
		if strings.Contains(lower, code) {
			return ReasonProviderAuth
		}
	}
	if strings.Contains(lower, "permission denied") ||
		strings.Contains(lower, "invalid api key") {
		return ReasonProviderAuth
	}
	if _, err := strconv.Atoi(lower); err == nil {
		return ReasonUnknown
	}
	return ReasonBusiness
}

func Decide(run Run) Decision {
	if run.Status == RunSucceeded {
		return Decision{
			ReviewRequired: true,
			IssueStatus:    "in_review",
			UserMessage:    "Run completed and is waiting for review.",
		}
	}
	reason := Classify(run.Error)
	switch reason {
	case ReasonProviderNetwork:
		return Decision{
			Reason:      reason,
			Retryable:   true,
			IssueStatus: "in_progress",
			UserMessage: "The provider connection failed; a bounded retry may continue the run.",
		}
	case ReasonProviderAuth:
		return Decision{
			Reason:         reason,
			ReviewRequired: true,
			IssueStatus:    "blocked",
			UserMessage:    "Provider access must be repaired before the run can continue.",
		}
	default:
		return Decision{
			Reason:         reason,
			ReviewRequired: true,
			IssueStatus:    "blocked",
			UserMessage:    "The run failed and needs review.",
		}
	}
}
~~~

Classify 先匹配更具体的网络和权限证据，再把剩余非空错误归入业务失败。Decide 把分类结果与 Run 是否成功组合成策略结果。注意成功路径返回 in_review，它没有返回 shipped。

真实源码对照 server/pkg/taskfailure/classify.go:Classify。真实分类器拥有更完整的 provider、runtime、context overflow 和 process failure 规则。教程参考项目只保留本版需要的三个决策类别，避免把分类规则数量误认为 Review 集成已经完成。

## 3. 冻结 V1

创建 v1/reference/review/failure_test.go：

~~~go
package review

import "testing"

func TestDecideSeparatesRetryFromReview(t *testing.T) {
	network := Decide(Run{Status: RunFailed, Error: "connection closed mid-response"})
	if !network.Retryable || network.ReviewRequired || network.Reason != ReasonProviderNetwork {
		t.Fatalf("network decision = %+v", network)
	}
	auth := Decide(Run{Status: RunFailed, Error: "HTTP 401 invalid api key"})
	if auth.Retryable || !auth.ReviewRequired || auth.IssueStatus != "blocked" {
		t.Fatalf("auth decision = %+v", auth)
	}
	success := Decide(Run{Status: RunSucceeded})
	if !success.ReviewRequired || success.IssueStatus != "in_review" {
		t.Fatalf("success decision = %+v", success)
	}
}

func TestUnknownFailureDoesNotBecomeRetryable(t *testing.T) {
	decision := Decide(Run{Status: RunFailed, Error: "a business rule rejected the change"})
	if decision.Reason != ReasonBusiness || decision.Retryable || !decision.ReviewRequired {
		t.Fatalf("business decision = %+v", decision)
	}
}
~~~

运行：

~~~bash
cd tutorials/02-multica-delivery/modules/04-failure-review/v1/reference
go test ./...
~~~

预期两个测试通过。一个网络错误得到 Retryable=true 且不要求 Review，一个权限错误进入 blocked 并要求 Review，成功 Run 进入 in_review。这冻结了“自动继续”和“允许交付”分离的边界。

真实源码可对照 server/pkg/taskfailure/classify_test.go、server/internal/service/task_failure_event_test.go 和 server/internal/service/task_finalize_failure_test.go。下一版把 Retryable 变成带次数、延迟和 lineage 保留的 retry child 计划。
