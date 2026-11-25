# Multica Runtime Access v1：先把 Runtime 状态变成统一 verdict

[V0 设计](../V0-design.md) · [模块路线图](../plan.md) · [本版计划](plan.md) · [下一版 v2](../v2/access-policy.md)

`RunRequest` 带有 RuntimeID 时，服务不能直接假设该 Runtime 存在且可执行。真实 Multica 的 `RuntimeLookup` 区分找不到、数据库错误和成功读取，`AgentReadiness` 再把 runtime 状态分成 available、waitable 和 blocked。v1 先建立这条纯状态链，不引入权限和进程副作用。

真实入口是 `server/internal/service/agent_runtime_lookup.go::RuntimeLookup` 和 `agent_ready.go::AgentReadiness`。参考项目使用内存 Registry，保持同样的输入输出边界。

## 1. 创建 Go 参考项目

```bash
mkdir -p tutorials/02-multica-delivery/modules/02-runtime-access/v1/reference/runtime
cd tutorials/02-multica-delivery/modules/02-runtime-access/v1/reference
go mod init example.com/multica-runtime-access
```

## 2. 定义 Runtime 与 lookup

创建 `v1/reference/runtime/model.go`：

```go
package runtime

type Status string

const (
	Online   Status = "online"
	Offline  Status = "offline"
	Unusable Status = "unusable"
)

type Runtime struct {
	ID          string
	WorkspaceID string
	OwnerID     string
	Visibility  string
	SharedWith  []string
	Status      Status
}

type Agent struct {
	ID        string
	OwnerID   string
	RuntimeID string
}

type RunRequest struct {
	ID          string
	WorkspaceID string
	AgentID     string
	RuntimeID   string
	Prompt      string
}

type Registry struct {
	runtimes map[string]Runtime
}

func NewRegistry(runtimes ...Runtime) *Registry {
	result := &Registry{runtimes: map[string]Runtime{}}
	for _, runtime := range runtimes {
		result.runtimes[runtime.ID] = runtime
	}
	return result
}

func (r *Registry) Get(runtimeID string) (Runtime, bool) {
	runtime, ok := r.runtimes[runtimeID]
	return runtime, ok
}
```

Registry 只保存 Runtime 事实。它不判断调用者，也不启动 runner。`RunRequest` 同时带 workspace、agent 和 runtime identity，后续 policy 可以检查它们是否属于同一控制范围。

## 3. 实现 readiness verdict

创建 `v1/reference/runtime/readiness.go`：

```go
package runtime

type Availability string

const (
	Ready    Availability = "ready"
	Waitable Availability = "waitable"
	Blocked  Availability = "blocked"
	Unknown  Availability = "unknown"
)

type Verdict struct {
	Availability Availability
	Reason       string
	Runtime      Runtime
}

func Readiness(agent Agent, registry *Registry) Verdict {
	if agent.RuntimeID == "" {
		return Verdict{Availability: Blocked, Reason: "runtime_required"}
	}
	runtime, ok := registry.Get(agent.RuntimeID)
	if !ok {
		return Verdict{Availability: Unknown, Reason: "runtime_not_found"}
	}
	switch runtime.Status {
	case Online:
		return Verdict{Availability: Ready, Runtime: runtime}
	case Offline:
		return Verdict{Availability: Waitable, Reason: "runtime_offline", Runtime: runtime}
	case Unusable:
		return Verdict{Availability: Blocked, Reason: "runtime_unusable", Runtime: runtime}
	default:
		return Verdict{Availability: Unknown, Reason: "runtime_status_unknown", Runtime: runtime}
	}
}
```

`Unknown` 代表服务没有足够事实作出执行判断。它不能被调用方当作 Ready。offline 的 waitable verdict 交给 Scheduler 处理，adapter 在 v3 才会看到 Ready 请求。

## 4. 冻结 V1

创建 `v1/reference/runtime/readiness_test.go`：

```go
package runtime

import "testing"

func TestReadinessSeparatesRuntimeStates(t *testing.T) {
	agent := Agent{ID: "agent-1", RuntimeID: "rt-1"}
	for _, test := range []struct {
		name string
		status Status
		want Availability
	}{
		{"online", Online, Ready},
		{"offline", Offline, Waitable},
		{"unusable", Unusable, Blocked},
	} {
		t.Run(test.name, func(t *testing.T) {
			verdict := Readiness(agent, NewRegistry(Runtime{ID: "rt-1", Status: test.status}))
			if verdict.Availability != test.want {
				t.Fatalf("verdict = %+v, want %s", verdict, test.want)
			}
		})
	}
}

func TestMissingOrUnboundRuntimeDoesNotBecomeReady(t *testing.T) {
	if got := Readiness(Agent{}, NewRegistry()); got.Availability != Blocked {
		t.Fatalf("unbound agent = %+v, want blocked", got)
	}
	if got := Readiness(Agent{RuntimeID: "missing"}, NewRegistry()); got.Availability != Unknown {
		t.Fatalf("missing runtime = %+v, want unknown", got)
	}
}
```

执行：

```bash
cd tutorials/02-multica-delivery/modules/02-runtime-access/v1/reference
go test ./...
```

预期所有测试通过。这个检查只证明 Runtime facts 到 verdict 的映射，不证明 workspace access 或进程启动。v2 接入 `RuntimeAllowsUser`。

## 检查点与下一步

v1 建立了一个不会把未知状态当成 ready 的 lookup 边界。下一版将加入 owner、public、private shared 和 workspace 作用域检查。
