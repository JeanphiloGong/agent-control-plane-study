# Multica Issue / Invocation v2：把 Runtime readiness 接到 Run 请求

[上一版 v1](../v1/issue-invocation.md) · [本版计划](plan.md) · [下一版 v3](../v3/run-result.md)

权限通过只说明“谁可以请求 Agent”，不说明 Agent 当前能否执行。真实 Multica 的 `AgentReadiness` 区分三种结果。online 可以立即进入执行，普通 offline 可以排队等待，缺少 runtime、归档 agent、权限拒绝和不可修复 runtime 必须阻止触发。

## 1. 增加 Runtime 和 readiness verdict

创建 `v1/reference/issue/readiness.go`：

```go
package issue

type RuntimeStatus string

const (
	RuntimeOnline   RuntimeStatus = "online"
	RuntimeOffline  RuntimeStatus = "offline"
	RuntimeUnusable RuntimeStatus = "unusable"
)

type Runtime struct {
	ID     string
	Status RuntimeStatus
}

type RuntimeRegistry map[string]Runtime

type Availability string

const (
	Available Availability = "available"
	Waitable  Availability = "waitable"
	Blocked   Availability = "blocked"
)

type ReadinessReason string

const (
	ReasonNone            ReadinessReason = ""
	ReasonAgentArchived   ReadinessReason = "target_unavailable"
	ReasonRuntimeRequired ReadinessReason = "agent_runtime_required"
	ReasonRuntimeOffline  ReadinessReason = "runtime_offline"
	ReasonRuntimeUnusable ReadinessReason = "runtime_unusable"
)

type Verdict struct {
	Availability Availability
	Reason       ReadinessReason
	Detail       string
}

func (v Verdict) Ready() bool   { return v.Availability == Available }
func (v Verdict) Waits() bool   { return v.Availability == Waitable }
func (v Verdict) Blocked() bool { return v.Availability == Blocked }

func AgentReadiness(agent Agent, runtimes RuntimeRegistry) Verdict {
	if agent.Archived {
		return Verdict{Availability: Blocked, Reason: ReasonAgentArchived, Detail: "agent is archived"}
	}
	if agent.RuntimeID == "" {
		return Verdict{Availability: Blocked, Reason: ReasonRuntimeRequired, Detail: "agent has no runtime bound"}
	}
	runtime, ok := runtimes[agent.RuntimeID]
	if !ok {
		return Verdict{Availability: Blocked, Reason: ReasonRuntimeRequired, Detail: "runtime was not found"}
	}
	switch runtime.Status {
	case RuntimeOnline:
		return Verdict{Availability: Available}
	case RuntimeOffline:
		return Verdict{Availability: Waitable, Reason: ReasonRuntimeOffline, Detail: "runtime is offline and can return"}
	case RuntimeUnusable:
		return Verdict{Availability: Blocked, Reason: ReasonRuntimeUnusable, Detail: "runtime needs repair"}
	default:
		return Verdict{Availability: Blocked, Reason: ReasonRuntimeUnusable, Detail: "runtime status is unknown"}
	}
}
```

`Waitable` 是一个真实业务结果，不是 Online 的别名。它告诉下游调度器可以保留请求，等 Runtime 恢复后再 claim。`Blocked` 才表示这次 invocation 需要调用方处理。

## 2. 扩展 RunRequest 与 Service

在 `v1/reference/issue/model.go` 的 `RunRequest` 中加入：

```go
	Deferred bool
	Reason   ReadinessReason
```

将 `v1/reference/issue/service.go` 的 `Service`、`NewService` 和 `Create` 替换为：

```go
type Service struct {
	issues   map[string]Issue
	requests map[string]RunRequest
	runtimes RuntimeRegistry
	nextID   int
}

func NewService(runtimes RuntimeRegistry) *Service {
	return &Service{issues: map[string]Issue{}, requests: map[string]RunRequest{}, runtimes: runtimes}
}

func (s *Service) Create(input CreateIssueInput) (CreateResult, error) {
	if input.WorkspaceID == "" || input.Title == "" {
		return CreateResult{}, errors.New("workspace and title are required")
	}
	var verdict Verdict
	if input.Agent != nil {
		if !CanMemberInvokeAgent(input.Creator, input.WorkspaceID, *input.Agent) {
			return CreateResult{}, errors.New("agent invocation is not allowed")
		}
		verdict = AgentReadiness(*input.Agent, s.runtimes)
		if verdict.Blocked() {
			return CreateResult{}, fmt.Errorf("agent is not ready: %s", verdict.Reason)
		}
	}

	s.nextID++
	issue := Issue{ID: fmt.Sprintf("issue-%d", s.nextID), WorkspaceID: input.WorkspaceID, Title: input.Title, Status: IssueTodo}
	if input.Agent != nil {
		issue.AssigneeAgentID = input.Agent.ID
	}
	s.issues[issue.ID] = issue
	result := CreateResult{Issue: issue}
	if input.Agent != nil {
		run := RunRequest{ID: fmt.Sprintf("run-%d", s.nextID), IssueID: issue.ID, AgentID: input.Agent.ID, RuntimeID: input.Agent.RuntimeID, Prompt: input.Title, Deferred: verdict.Waits(), Reason: verdict.Reason}
		s.requests[run.ID] = run
		result.Run = &run
	}
	return result, nil
}
```

这里先做权限，再做 readiness，最后才写 Issue 和 RunRequest。blocked readiness 不会留下半成品。offline 会把 `Deferred` 设为 true，后续 Scheduler 可以识别它。v2 仍然不启动 runner。

## 3. 冻结 V2

创建 `v1/reference/issue/readiness_test.go`：

```go
package issue

import "testing"

func readyAgent() Agent { return Agent{ID: "agent-1", OwnerID: "u-1", PermissionMode: "private", RuntimeID: "rt-1"} }

func TestReadinessAndCreateOutcomes(t *testing.T) {
	tests := []struct {
		name string
		runtime RuntimeRegistry
		want Availability
		wantError bool
		wantDeferred bool
	}{
		{"online", RuntimeRegistry{"rt-1": {ID: "rt-1", Status: RuntimeOnline}}, Available, false, false},
		{"offline", RuntimeRegistry{"rt-1": {ID: "rt-1", Status: RuntimeOffline}}, Waitable, false, true},
		{"missing", RuntimeRegistry{}, Blocked, true, false},
		{"unusable", RuntimeRegistry{"rt-1": {ID: "rt-1", Status: RuntimeUnusable}}, Blocked, true, false},
	}
	for _, test := range tests {
		t.Run(test.name, func(t *testing.T) {
			agent := readyAgent()
			verdict := AgentReadiness(agent, test.runtime)
			if verdict.Availability != test.want {
				t.Fatalf("verdict = %+v, want %s", verdict, test.want)
			}
			service := NewService(test.runtime)
			result, err := service.Create(CreateIssueInput{WorkspaceID: "ws-1", Title: test.name, Creator: owner(), Agent: &agent})
			if (err != nil) != test.wantError {
				t.Fatalf("err = %v, want error=%v", err, test.wantError)
			}
			if !test.wantError && result.Run.Deferred != test.wantDeferred {
				t.Fatalf("run = %+v, want deferred=%v", result.Run, test.wantDeferred)
			}
			if test.wantError && service.IssueCount() != 0 {
				t.Fatalf("blocked readiness wrote %d issues", service.IssueCount())
			}
		})
	}
}
```

把 v1 测试中的 `NewService()` 调用更新为带 online registry 的 `NewService(RuntimeRegistry{"rt-1": {ID: "rt-1", Status: RuntimeOnline}})`，并给 owner/private 测试 Agent 加上 `RuntimeID: "rt-1"`。这是 readiness 接入后的必要输入变化。再从参考项目执行：

```bash
cd tutorials/02-multica-delivery/modules/01-issue-invocation/v1/reference
go test ./...
```

预期是 v1 与 v2 测试全部通过。v2 的能力冻结是 readiness 与 invocation 的连接，claim/lease 仍由下一个模块负责。

## 检查点与下一步

当前 RunRequest 已经包含 AgentID、RuntimeID、Deferred 和 readiness reason。下一版会让 runner 返回成功或失败，并把结果写回 Issue，保持 review 状态独立。
