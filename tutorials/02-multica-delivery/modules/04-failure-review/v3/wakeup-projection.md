# Multica Failure / Review v3：Issue Wakeup 与状态投影

[上一版 v2](../v2/retry-policy.md) · [本版计划](plan.md) · [模块路线图](../plan.md)

V2 生成了 retry child，但 deferred child 仍需要一个受条件控制的入口。Issue Wakeup 会在规则到期、Issue 仍满足条件时创建下一次 Run，并消费一次性规则。客户端投影只读取服务端决策，不能自己把成功 Run 改成 shipped。

## 1. 实现受条件和去重保护的 Wakeup

创建 v1/reference/review/wakeup.go：

~~~go
package review

import (
	"sync"
	"time"
)

type WakeupService struct {
	mu    sync.Mutex
	rules map[string]WakeupRule
}

func NewWakeupService(rules ...WakeupRule) *WakeupService {
	store := make(map[string]WakeupRule, len(rules))
	for _, rule := range rules {
		store[rule.ID] = rule
	}
	return &WakeupService{rules: store}
}

func (s *WakeupService) Tick(now time.Time, issueStates map[string]string) []Run {
	s.mu.Lock()
	defer s.mu.Unlock()
	var runs []Run
	for id, rule := range s.rules {
		if !rule.Enabled || now.Before(rule.NextFireAt) ||
			issueStates[rule.IssueID] != rule.ExpectedState ||
			rule.LastDispatch == id {
			continue
		}
		runs = append(runs, Run{
			ID:           "wakeup/" + id,
			IssueID:      rule.IssueID,
			WorkflowStep: "wakeup",
			Handoff:      rule.Instruction,
			Status:       RunQueued,
			Attempt:      1,
			MaxAttempts:  1,
		})
		rule.LastDispatch = id
		rule.Enabled = false
		s.rules[id] = rule
	}
	return runs
}

func (s *WakeupService) Rule(id string) (WakeupRule, bool) {
	s.mu.Lock()
	defer s.mu.Unlock()
	rule, ok := s.rules[id]
	return rule, ok
}
~~~

Tick 同时检查 enabled、NextFireAt 和 Issue 当前状态。规则只 dispatch 一次，写入 LastDispatch 后关闭自身。它返回一个新的 Run，且保留 Issue ID 和 instruction 作为 handoff。

真实源码的 Tick 入口是 server/internal/service/issue_wakeup.go:IssueWakeupService.Tick。真实实现会在数据库事务中锁定 wakeup、检查条件、写 receipt，再进入 task enqueue。ListInputWakeups 和 ListDueWakeups 分开处理输入 lane 与 due lane，避免大量无关条件检查饿死到期规则。本参考代码只演示本版的条件与一次性去重规则。

## 2. 把服务端决策投影给 Issue

创建 v1/reference/review/projection.go：

~~~go
package review

func Project(run Run, decision Decision) IssueProjection {
	projection := IssueProjection{
		IssueID:      run.IssueID,
		Status:       decision.IssueStatus,
		RunID:        run.ID,
		Failure:      decision.Reason,
		ReviewNeeded: decision.ReviewRequired,
	}
	if run.Status == RunSucceeded {
		projection.Failure = ""
	}
	return projection
}
~~~

Project 只映射 Decision。成功结果是 in_review 且 ReviewNeeded=true。失败结果保留 failure reason 和 blocked/in_progress 等服务端状态。shipped 不在本版投影中，因为交付审核属于后续流程。

真实客户端的状态词在 packages/core/issues/config/status.ts，服务端 Run 事实仍由 task 和 Issue API 返回。客户端的 run-steering.ts 根据 task status 选择 starting、running、failed 等展示动作，不重新计算 retryability。

## 3. 冻结 V3

创建 v1/reference/review/wakeup_test.go：

~~~go
package review

import (
	"sync"
	"testing"
	"time"
)

func TestWakeupRequiresConditionAndConsumesOnce(t *testing.T) {
	now := time.Unix(100, 0).UTC()
	service := NewWakeupService(WakeupRule{
		ID: "wake-1", IssueID: "issue-1", Instruction: "continue delivery",
		ExpectedState: "blocked", Enabled: true, NextFireAt: now,
	})
	if runs := service.Tick(now, map[string]string{"issue-1": "in_progress"}); len(runs) != 0 {
		t.Fatalf("unmet condition dispatched %d runs", len(runs))
	}
	runs := service.Tick(now, map[string]string{"issue-1": "blocked"})
	if len(runs) != 1 || runs[0].Handoff != "continue delivery" {
		t.Fatalf("first dispatch = %+v", runs)
	}
	if runs := service.Tick(now.Add(time.Minute), map[string]string{"issue-1": "blocked"}); len(runs) != 0 {
		t.Fatalf("once wakeup dispatched again: %+v", runs)
	}
	rule, _ := service.Rule("wake-1")
	if rule.Enabled || rule.LastDispatch != "wake-1" {
		t.Fatalf("rule after dispatch = %+v", rule)
	}
}

func TestProjectionDoesNotTreatSuccessAsShipped(t *testing.T) {
	run := Run{ID: "run-3", IssueID: "issue-3", Status: RunSucceeded}
	decision := Decide(run)
	got := Project(run, decision)
	if got.Status != "in_review" || !got.ReviewNeeded || got.Status == "shipped" {
		t.Fatalf("projection = %+v", got)
	}
}

func TestConcurrentTicksDispatchOnce(t *testing.T) {
	now := time.Unix(100, 0).UTC()
	service := NewWakeupService(WakeupRule{
		ID: "wake-2", IssueID: "issue-2", ExpectedState: "blocked",
		Enabled: true, NextFireAt: now,
	})
	start := make(chan struct{})
	results := make(chan int, 2)
	var group sync.WaitGroup
	for range 2 {
		group.Add(1)
		go func() {
			defer group.Done()
			<-start
			results <- len(service.Tick(now, map[string]string{"issue-2": "blocked"}))
		}()
	}
	close(start)
	group.Wait()
	close(results)
	total := 0
	for count := range results {
		total += count
	}
	if total != 1 {
		t.Fatalf("concurrent ticks dispatched %d runs, want 1", total)
	}
}
~~~

运行：

~~~bash
cd tutorials/02-multica-delivery/modules/04-failure-review/v1/reference
go test ./...
go test -race ./...
~~~

预期七个测试全部通过。未满足状态条件时没有 dispatch，条件满足后只 dispatch 一次，并发 Tick 仍只产生一个 Run，成功 Run 仍停在 in_review。mutex 与竞态检测共同固定了参考规则的一次性边界。

真实源码的数据库验收入口包括 server/internal/service/issue_wakeup_test.go、issue_wakeup_tick_test.go、issue_wakeup_review_test.go 和 issue_wakeup_workflow_test.go。这些测试需要完整 Multica PostgreSQL fixture，教程只在本地 reference 中运行等价规则，不声称真实服务集成已经由本次回放通过。

## 完成边界

本版完成的是：

- 失败原因、自动重试和 Review 决策有稳定的数据流。
- retry child 保留 Issue/workflow handoff，并可通过 deferred 时间进入 Wakeup。
- Wakeup 按条件和一次性规则去重。
- 客户端看到 in_progress、blocked 或 in_review 等服务端事实。

本版没有实现完整数据库事务、所有 Wakeup 条件类型、Issue HTTP API 或 Review 审核操作。它们仍由 Multica 真实服务边界负责。
