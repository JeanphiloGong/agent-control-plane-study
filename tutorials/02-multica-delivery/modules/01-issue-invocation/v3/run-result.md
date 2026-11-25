# Multica Issue / Invocation v3：把 Run 结果回写 Issue，但保留 Review 边界

[上一版 v2](../v2/runtime-readiness.md) · [本版计划](plan.md) · [模块路线图](../plan.md)

RunRequest 生成后，实际执行属于后续 runtime/scheduler。为了让本模块形成闭环，参考项目加入一个注入式 Runner。Runner 返回成功或失败，Service 负责状态转移和 Issue 回写。Runner 不直接修改 Issue，Review 也不在这里被批准。

真实源码的 Issue 创建会调用 TaskService enqueue，后续 Task/Runtime/Scheduler 负责执行。这里的 `Execute` 只是把那个交接边界压缩成一个可回放的同步调用。

## 1. 扩展状态模型

在 `v1/reference/issue/model.go` 中加入这些类型和字段：

```go
type RunStatus string

const (
	RunQueued    RunStatus = "queued"
	RunRunning   RunStatus = "running"
	RunSucceeded RunStatus = "succeeded"
	RunFailed    RunStatus = "failed"
)

type ReviewStatus string

const ReviewPending ReviewStatus = "pending"
```

给 `Issue` 增加：

```go
	RunStatus    RunStatus
	ReviewStatus ReviewStatus
```

给 `RunRequest` 增加：

```go
	Status  RunStatus
	Summary string
	Error   string
```

在 v2 `Service.Create` 创建 Issue 时把 `RunStatus` 和 `ReviewStatus` 初始化为 `RunQueued` 与 `ReviewPending`，创建 RunRequest 时把 `Status` 初始化为 `RunQueued`。没有 Agent 的普通 Issue 可以保持 `RunStatus` 为空。

将 v2 `Create` 中的两处构造替换为：

```go
issue := Issue{
	ID:              fmt.Sprintf("issue-%d", s.nextID),
	WorkspaceID:     input.WorkspaceID,
	Title:           input.Title,
	Status:          IssueTodo,
	ReviewStatus:    ReviewPending,
}
if input.Agent != nil {
	issue.AssigneeAgentID = input.Agent.ID
}
```

```go
run := RunRequest{
	ID:        fmt.Sprintf("run-%d", s.nextID),
	IssueID:   issue.ID,
	AgentID:   input.Agent.ID,
	RuntimeID: input.Agent.RuntimeID,
	Prompt:    input.Title,
	Deferred:  verdict.Waits(),
	Reason:    verdict.Reason,
	Status:    RunQueued,
}
```

## 2. 添加 Runner 结果和状态转移

创建 `v1/reference/issue/runner.go`：

```go
package issue

type Outcome struct {
	Succeeded bool
	Summary   string
	Error     string
}

type Runner interface {
	Run(RunRequest) Outcome
}

type FakeRunner struct {
	Outcome Outcome
}

func (r FakeRunner) Run(RunRequest) Outcome { return r.Outcome }
```

在 `v1/reference/issue/service.go` 末尾加入：

```go
func (s *Service) StartRun(runID string) (RunRequest, error) {
	run, ok := s.requests[runID]
	if !ok {
		return RunRequest{}, errors.New("run not found")
	}
	if run.Status != RunQueued {
		return RunRequest{}, errors.New("run is not queued")
	}
	run.Status = RunRunning
	s.requests[runID] = run
	issue := s.issues[run.IssueID]
	issue.RunStatus = RunRunning
	s.issues[run.IssueID] = issue
	return run, nil
}

func (s *Service) CompleteRun(runID string, outcome Outcome) error {
	run, ok := s.requests[runID]
	if !ok {
		return errors.New("run not found")
	}
	if run.Status != RunRunning {
		return errors.New("run is not running")
	}
	run.Summary = outcome.Summary
	run.Error = outcome.Error
	if outcome.Succeeded {
		run.Status = RunSucceeded
	} else {
		run.Status = RunFailed
	}
	s.requests[runID] = run
	issue := s.issues[run.IssueID]
	if outcome.Succeeded {
		issue.Status = IssueStatus("done")
	} else {
		issue.Status = IssueStatus("blocked")
	}
	issue.RunStatus = run.Status
	issue.ReviewStatus = ReviewPending
	s.issues[issue.ID] = issue
	return nil
}

func (s *Service) Execute(runID string, runner Runner) error {
	run, err := s.StartRun(runID)
	if err != nil {
		return err
	}
	return s.CompleteRun(runID, runner.Run(run))
}

func (s *Service) GetIssue(issueID string) (Issue, bool) {
	issue, ok := s.issues[issueID]
	return issue, ok
}
```

`StartRun` 负责 queued → running，`CompleteRun` 只接受 running。成功和失败都把 review 留在 pending，避免把“执行结束”提升成“审核通过”。

## 3. 冻结 V3

创建 `v1/reference/issue/run_test.go`：

```go
package issue

import "testing"

func onlineService() *Service {
	return NewService(RuntimeRegistry{"rt-1": {ID: "rt-1", Status: RuntimeOnline}})
}

func createRun(t *testing.T, service *Service) string {
	t.Helper()
	agent := Agent{ID: "agent-1", OwnerID: "u-1", PermissionMode: "private", RuntimeID: "rt-1"}
	result, err := service.Create(CreateIssueInput{WorkspaceID: "ws-1", Title: "Ship", Creator: owner(), Agent: &agent})
	if err != nil {
		t.Fatal(err)
	}
	return result.Run.ID
}

func TestExecuteSuccessKeepsReviewPending(t *testing.T) {
	service := onlineService()
	runID := createRun(t, service)
	if err := service.Execute(runID, FakeRunner{Outcome: Outcome{Succeeded: true, Summary: "tests passed"}}); err != nil {
		t.Fatal(err)
	}
	issue, _ := service.GetIssue("issue-1")
	if issue.Status != IssueStatus("done") || issue.RunStatus != RunSucceeded {
		t.Fatalf("issue = %+v, want done/succeeded", issue)
	}
	if issue.ReviewStatus != ReviewPending {
		t.Fatalf("review = %q, want pending", issue.ReviewStatus)
	}
}

func TestExecuteFailureBlocksIssueAndRejectsSecondCompletion(t *testing.T) {
	service := onlineService()
	runID := createRun(t, service)
	if err := service.Execute(runID, FakeRunner{Outcome: Outcome{Error: "provider failed"}}); err != nil {
		t.Fatal(err)
	}
	issue, _ := service.GetIssue("issue-1")
	if issue.Status != IssueStatus("blocked") || issue.RunStatus != RunFailed {
		t.Fatalf("issue = %+v, want blocked/failed", issue)
	}
	if err := service.CompleteRun(runID, Outcome{Succeeded: true}); err == nil {
		t.Fatal("second completion unexpectedly succeeded")
	}
}
```

从 `v1/reference` 执行最终回放：

```bash
cd tutorials/02-multica-delivery/modules/01-issue-invocation/v1/reference
go test ./...
```

预期是 v1、v2、v3 的测试全部通过。这个版本完成了 Issue Invocation 的局部闭环，但没有实现 Scheduler claim、lease token、heartbeat、retry 或 Review Gate。

## 检查点与下一模块

`01-issue-invocation` 现在交付了授权、readiness、RunRequest 和结果回写。下一个模块是 [02 Runtime Access](../../02-runtime-access/V0-design.md)，负责把 Runtime lookup、ownership/access policy 和 runner adapter 从 invocation 服务中拆出来。
