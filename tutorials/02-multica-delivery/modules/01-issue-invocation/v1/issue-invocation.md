# Multica Issue / Invocation v1：权限通过后生成一次 Run 请求

[V0 设计](../V0-design.md) · [模块路线图](../plan.md) · [本版计划](plan.md) · [下一版 v2](../v2/runtime-readiness.md)

用户创建一个 Issue 并把它分配给 Agent。服务必须先回答“这个成员能否调用这个 Agent”，再决定是否创建 Issue 和 RunRequest。权限失败不能先写 Issue 再补偿删除，因为下游会把已经提交的 Issue 看成可执行工作。

真实 Multica 的入口是 `server/internal/service/issue.go::IssueService.Create`，授权函数是 `agent_invocation.go::CanMemberInvokeAgent`。真实服务还负责事务、编号、标签、事件和数据库错误，本参考项目先冻结 invocation 的核心边界。

## 1. 创建 Go 参考项目

从仓库根目录执行：

```bash
mkdir -p tutorials/02-multica-delivery/modules/01-issue-invocation/v1/reference/issue
cd tutorials/02-multica-delivery/modules/01-issue-invocation/v1/reference
go mod init example.com/multica-issue-invocation
```

## 2. 定义领域输入和输出

创建 `v1/reference/issue/model.go`：

```go
package issue

type IssueStatus string

const IssueTodo IssueStatus = "todo"

type InvocationTarget struct {
	Kind string
	ID   string
}

type Agent struct {
	ID                string
	OwnerID           string
	PermissionMode    string
	InvocationTargets []InvocationTarget
	RuntimeID         string
	Archived          bool
}

type Member struct {
	UserID      string
	WorkspaceID string
	Role        string
}

type Issue struct {
	ID              string
	WorkspaceID     string
	Title           string
	Status          IssueStatus
	AssigneeAgentID string
}

type RunRequest struct {
	ID        string
	IssueID   string
	AgentID   string
	RuntimeID string
	Prompt    string
}

type CreateIssueInput struct {
	WorkspaceID string
	Title       string
	Creator     Member
	Agent       *Agent
}

type CreateResult struct {
	Issue Issue
	Run   *RunRequest
}
```

`Issue` 是持久化事实的最小投影，`RunRequest` 是交给后续调度模块的执行意图。V1 的 `RuntimeID` 可以为空，因为 readiness 还没有接入。`Agent` 保留 runtime 字段，让 v2 可以在同一个模型上继续。

## 3. 实现 invocation policy

创建 `v1/reference/issue/policy.go`：

```go
package issue

// CanMemberInvokeAgent applies the invocation boundary before any Issue write.
func CanMemberInvokeAgent(member Member, workspaceID string, agent Agent) bool {
	if member.UserID == "" || member.WorkspaceID != workspaceID || member.Role == "guest" {
		return false
	}
	if member.UserID == agent.OwnerID {
		return true
	}
	if agent.PermissionMode != "public_to" {
		return false
	}
	for _, target := range agent.InvocationTargets {
		switch target.Kind {
		case "workspace":
			if target.ID == workspaceID {
				return true
			}
		case "member":
			if target.ID == member.UserID {
				return true
			}
		}
	}
	return false
}
```

owner 直接通过。`public_to` 仍需要明确的 workspace 或 member target，guest 永远不能触发 Agent。这个函数只返回权限事实，不创建进程、不写 Issue，也不读取 Runtime。

## 4. 让服务编排写入顺序

创建 `v1/reference/issue/service.go`：

```go
package issue

import (
	"errors"
	"fmt"
)

type Service struct {
	issues   map[string]Issue
	requests map[string]RunRequest
	nextID   int
}

func NewService() *Service {
	return &Service{issues: map[string]Issue{}, requests: map[string]RunRequest{}}
}

func (s *Service) Create(input CreateIssueInput) (CreateResult, error) {
	if input.WorkspaceID == "" || input.Title == "" {
		return CreateResult{}, errors.New("workspace and title are required")
	}
	if input.Agent != nil && !CanMemberInvokeAgent(input.Creator, input.WorkspaceID, *input.Agent) {
		return CreateResult{}, errors.New("agent invocation is not allowed")
	}

	s.nextID++
	issue := Issue{ID: fmt.Sprintf("issue-%d", s.nextID), WorkspaceID: input.WorkspaceID, Title: input.Title, Status: IssueTodo}
	if input.Agent != nil {
		issue.AssigneeAgentID = input.Agent.ID
	}
	s.issues[issue.ID] = issue

	result := CreateResult{Issue: issue}
	if input.Agent != nil {
		run := RunRequest{ID: fmt.Sprintf("run-%d", s.nextID), IssueID: issue.ID, AgentID: input.Agent.ID, Prompt: input.Title}
		s.requests[run.ID] = run
		result.Run = &run
	}
	return result, nil
}

func (s *Service) IssueCount() int { return len(s.issues) }

func (s *Service) RequestCount() int { return len(s.requests) }
```

权限判断发生在 `issues` 写入前。服务只保存一个内存事实，后续版本会把 RunRequest 的 RuntimeID 和状态补齐。这个参考项目没有伪造事务，真实 `IssueService.Create` 的事务边界仍以 Go 服务源码为准。

## 5. 冻结 V1

创建 `v1/reference/issue/service_test.go`：

```go
package issue

import "testing"

func owner() Member { return Member{UserID: "u-1", WorkspaceID: "ws-1", Role: "member"} }

func publicAgent() Agent {
	return Agent{ID: "agent-1", OwnerID: "u-owner", PermissionMode: "public_to", InvocationTargets: []InvocationTarget{{Kind: "workspace", ID: "ws-1"}}}
}

func TestOwnerInvocationCreatesIssueAndRun(t *testing.T) {
	agent := Agent{ID: "agent-1", OwnerID: "u-1", PermissionMode: "private"}
	result, err := NewService().Create(CreateIssueInput{WorkspaceID: "ws-1", Title: "Fix bug", Creator: owner(), Agent: &agent})
	if err != nil {
		t.Fatal(err)
	}
	if result.Run == nil || result.Run.AgentID != agent.ID {
		t.Fatalf("run = %+v, want agent invocation", result.Run)
	}
}

func TestWorkspaceTargetAllowsMember(t *testing.T) {
	agent := publicAgent()
	result, err := NewService().Create(CreateIssueInput{WorkspaceID: "ws-1", Title: "Review", Creator: owner(), Agent: &agent})
	if err != nil || result.Run == nil {
		t.Fatalf("result = %+v, err = %v", result, err)
	}
}

func TestGuestAndUnlistedMemberAreRejectedBeforeWrites(t *testing.T) {
	service := NewService()
	agent := publicAgent()
	guest := Member{UserID: "u-guest", WorkspaceID: "ws-1", Role: "guest"}
	if _, err := service.Create(CreateIssueInput{WorkspaceID: "ws-1", Title: "Denied", Creator: guest, Agent: &agent}); err == nil {
		t.Fatal("guest invocation unexpectedly succeeded")
	}
	other := Member{UserID: "u-other", WorkspaceID: "ws-1", Role: "member"}
	private := Agent{ID: "private", OwnerID: "u-owner", PermissionMode: "private"}
	if _, err := service.Create(CreateIssueInput{WorkspaceID: "ws-1", Title: "Denied", Creator: other, Agent: &private}); err == nil {
		t.Fatal("unlisted invocation unexpectedly succeeded")
	}
	if service.IssueCount() != 0 || service.RequestCount() != 0 {
		t.Fatalf("denied invocation wrote state: issues=%d requests=%d", service.IssueCount(), service.RequestCount())
	}
}
```

从 `v1/reference` 执行：

```bash
cd tutorials/02-multica-delivery/modules/01-issue-invocation/v1/reference
go test ./...
```

预期是 3 个 Go 测试通过。V1 冻结“权限通过才有 Issue/RunRequest 写入”。它还没有判断 Runtime 是否在线，下一版会在相同 service 边界接入 readiness。

## 检查点与下一步

V1 的 RunRequest 是可审计的执行意图，后续 Scheduler 可以用它生成自己的 claim。下一版不会把权限函数改成启动器，而是增加 Runtime registry 和 AgentReadiness，继续在 runner 之前阻止不可执行请求。
