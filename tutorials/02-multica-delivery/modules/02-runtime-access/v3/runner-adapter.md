# Multica Runtime Access v3：让 adapter 只接收通过边界的请求

[上一版 v2](../v2/access-policy.md) · [本版计划](plan.md) · [模块路线图](../plan.md)

Runtime registry 和 access policy 都是纯状态判断。真正启动 Agent 的副作用属于 adapter。服务层必须把 Ready verdict 转换为 `StartRequest`，把 offline 转换为 deferred，把 denied/unknown 直接返回，并把 adapter 的错误保持为启动错误。

## 1. 定义 adapter contract

创建 `v1/reference/runtime/adapter.go`：

```go
package runtime

type StartRequest struct {
	RunID       string
	WorkspaceID string
	AgentID     string
	Runtime     Runtime
	Prompt      string
}

type StartHandle struct {
	ProcessID string
}

type StartResult struct {
	Started  bool
	Deferred bool
	Handle   StartHandle
	Reason   string
}

type RunnerAdapter interface {
	Start(StartRequest) (StartHandle, error)
}

type FakeAdapter struct {
	Calls []StartRequest
	Err   error
}

func (a *FakeAdapter) Start(request StartRequest) (StartHandle, error) {
	a.Calls = append(a.Calls, request)
	if a.Err != nil {
		return StartHandle{}, a.Err
	}
	return StartHandle{ProcessID: "process-1"}, nil
}
```

`RunnerAdapter` 只负责启动边界。FakeAdapter 记录 calls，让测试能证明拒绝和 deferred 分支没有外部副作用。

## 2. 编排 AccessService

创建 `v1/reference/runtime/service.go`：

```go
package runtime

import "fmt"

type AccessService struct {
	Registry *Registry
}

func (s *AccessService) Start(request RunRequest, member Member, agent Agent, adapter RunnerAdapter) (StartResult, error) {
	verdict := Authorize(request, member, agent, s.Registry)
	if verdict.Availability != AccessAllowed {
		return StartResult{Reason: verdict.Reason}, fmt.Errorf("runtime access denied: %s", verdict.Reason)
	}
	if verdict.Readiness.Availability == Waitable {
		return StartResult{Deferred: true, Reason: verdict.Readiness.Reason}, nil
	}
	if verdict.Readiness.Availability != Ready {
		return StartResult{Reason: verdict.Readiness.Reason}, fmt.Errorf("runtime is not ready: %s", verdict.Readiness.Reason)
	}
	handle, err := adapter.Start(StartRequest{RunID: request.ID, WorkspaceID: request.WorkspaceID, AgentID: agent.ID, Runtime: verdict.Runtime, Prompt: request.Prompt})
	if err != nil {
		return StartResult{Reason: "adapter_start_failed"}, err
	}
	return StartResult{Started: true, Handle: handle}, nil
}
```

这个服务只把 Ready + Allowed 请求交给 adapter。它不修改 Issue，不把启动成功写成 Run done，也不接管 Scheduler 的 claim。adapter 的错误被返回给上游分类器。

## 3. 冻结 V3

创建 `v1/reference/runtime/service_test.go`：

```go
package runtime

import (
	"errors"
	"testing"
)

func runtimeService(status Status) (*AccessService, RunRequest, Member, Agent, *FakeAdapter) {
	runtime := Runtime{ID: "rt-1", WorkspaceID: "ws-1", OwnerID: "owner", Visibility: "private", Status: status}
	return &AccessService{Registry: NewRegistry(runtime)}, RunRequest{ID: "run-1", WorkspaceID: "ws-1", RuntimeID: "rt-1", Prompt: "run"}, Member{UserID: "owner", WorkspaceID: "ws-1"}, Agent{ID: "agent-1", RuntimeID: "rt-1"}, &FakeAdapter{}
}

func TestReadyStartsAdapter(t *testing.T) {
	service, request, member, agent, adapter := runtimeService(Online)
	result, err := service.Start(request, member, agent, adapter)
	if err != nil || !result.Started || len(adapter.Calls) != 1 {
		t.Fatalf("result = %+v, err = %v, calls = %d", result, err, len(adapter.Calls))
	}
}

func TestOfflineDefersWithoutStartingAdapter(t *testing.T) {
	service, request, member, agent, adapter := runtimeService(Offline)
	result, err := service.Start(request, member, agent, adapter)
	if err != nil || !result.Deferred || len(adapter.Calls) != 0 {
		t.Fatalf("result = %+v, err = %v, calls = %d", result, err, len(adapter.Calls))
	}
}

func TestDeniedAndAdapterFailureHaveNoSuccessSideEffect(t *testing.T) {
	service, request, member, agent, adapter := runtimeService(Online)
	member.UserID = "other"
	if _, err := service.Start(request, member, agent, adapter); err == nil || len(adapter.Calls) != 0 {
		t.Fatal("denied request reached adapter")
	}
	member.UserID = "owner"
	adapter.Err = errors.New("runner unavailable")
	result, err := service.Start(request, member, agent, adapter)
	if err == nil || result.Started || result.Reason != "adapter_start_failed" || len(adapter.Calls) != 1 {
		t.Fatalf("result = %+v, err = %v, calls = %d", result, err, len(adapter.Calls))
	}
}
```

从 `v1/reference` 执行最终回放：

```bash
cd tutorials/02-multica-delivery/modules/02-runtime-access/v1/reference
go test ./...
```

预期 v1–v3 的全部测试通过。它证明 adapter 的调用边界和 deferred/error 结果，不证明真实 Runtime daemon、Agent 进程或 Scheduler lease。

## 检查点与下一模块

Runtime Access v3 已经把 Runtime record、access policy、readiness 和 runner adapter 串成一条可观察路径。下一模块会追踪 Multica 怎样领取系统任务的到期计划，以及 runner 如何续期、接管和提交终态。 [03 Scheduler / Lease](../../03-scheduler/V0-design.md)
