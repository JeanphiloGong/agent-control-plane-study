# Multica Runtime Access v2：把 Runtime ownership 放在启动之前

[上一版 v1](../v1/runtime-registry.md) · [本版计划](plan.md) · [下一版 v3](../v3/runner-adapter.md)

Runtime online 仍然不等于当前成员有权使用它。真实 `RuntimeAllowsUser` 在 workspace membership 已经确认后，判断 owner、public 和 private shared。服务还必须检查 RunRequest workspace 与 Runtime workspace 一致，阻止跨 workspace identity 被拼接到启动请求。

## 1. 增加 Member 和访问 verdict

创建 `v1/reference/runtime/access.go`：

```go
package runtime

type Member struct {
	UserID      string
	WorkspaceID string
	Role        string
}

type AccessAvailability string

const (
	AccessAllowed AccessAvailability = "allowed"
	AccessDenied  AccessAvailability = "denied"
)

type AccessVerdict struct {
	Availability AccessAvailability
	Reason       string
	Readiness    Verdict
	Runtime      Runtime
}

func RuntimeAllowsUser(runtime Runtime, userID string) bool {
	if runtime.OwnerID == "" || userID == "" {
		return false
	}
	if runtime.OwnerID == userID || runtime.Visibility == "public" {
		return true
	}
	if runtime.Visibility != "private" {
		return false
	}
	for _, shared := range runtime.SharedWith {
		if shared == userID {
			return true
		}
	}
	return false
}

func Authorize(request RunRequest, member Member, agent Agent, registry *Registry) AccessVerdict {
	runtime, ok := registry.Get(request.RuntimeID)
	if !ok {
		return AccessVerdict{Availability: AccessDenied, Reason: "runtime_not_found"}
	}
	if runtime.WorkspaceID != request.WorkspaceID || member.WorkspaceID != request.WorkspaceID {
		return AccessVerdict{Availability: AccessDenied, Reason: "workspace_mismatch", Runtime: runtime}
	}
	if member.Role == "guest" || !RuntimeAllowsUser(runtime, member.UserID) {
		return AccessVerdict{Availability: AccessDenied, Reason: "runtime_access_denied", Runtime: runtime}
	}
	readiness := Readiness(agent, registry)
	return AccessVerdict{Availability: AccessAllowed, Readiness: readiness, Runtime: runtime}
}
```

`RuntimeAllowsUser` 不创建进程，也不决定 offline 是否可等待。`Authorize` 先守住 workspace/access 边界，再复用 v1 readiness。这样 owner/share policy 和状态 policy 可以分别测试。

## 2. 冻结 V2

创建 `v1/reference/runtime/access_test.go`：

```go
package runtime

import "testing"

func accessFixtures() (RunRequest, Agent, Registry) {
	return RunRequest{ID: "run-1", WorkspaceID: "ws-1", RuntimeID: "rt-1"}, Agent{ID: "agent-1", RuntimeID: "rt-1"}, *NewRegistry(Runtime{ID: "rt-1", WorkspaceID: "ws-1", OwnerID: "owner", Visibility: "private", SharedWith: []string{"shared"}, Status: Online})
}

func TestRuntimeAccessPolicy(t *testing.T) {
	request, agent, registry := accessFixtures()
	for _, test := range []struct {
		name string
		member Member
		want AccessAvailability
		reason string
	}{
		{"owner", Member{UserID: "owner", WorkspaceID: "ws-1"}, AccessAllowed, ""},
		{"shared", Member{UserID: "shared", WorkspaceID: "ws-1"}, AccessAllowed, ""},
		{"other", Member{UserID: "other", WorkspaceID: "ws-1"}, AccessDenied, "runtime_access_denied"},
		{"guest", Member{UserID: "shared", WorkspaceID: "ws-1", Role: "guest"}, AccessDenied, "runtime_access_denied"},
		{"workspace mismatch", Member{UserID: "owner", WorkspaceID: "ws-2"}, AccessDenied, "workspace_mismatch"},
	} {
		t.Run(test.name, func(t *testing.T) {
			verdict := Authorize(request, test.member, agent, &registry)
			if verdict.Availability != test.want || verdict.Reason != test.reason {
				t.Fatalf("verdict = %+v, want %s/%s", verdict, test.want, test.reason)
			}
		})
	}
}

func TestPublicRuntimeDoesNotIgnoreWorkspaceBoundary(t *testing.T) {
	runtime := Runtime{ID: "rt-1", WorkspaceID: "ws-1", OwnerID: "owner", Visibility: "public", Status: Online}
	registry := NewRegistry(runtime)
	request := RunRequest{WorkspaceID: "ws-2", RuntimeID: runtime.ID}
	verdict := Authorize(request, Member{UserID: "owner", WorkspaceID: "ws-2"}, Agent{RuntimeID: runtime.ID}, registry)
	if verdict.Availability != AccessDenied || verdict.Reason != "workspace_mismatch" {
		t.Fatalf("verdict = %+v, want workspace mismatch", verdict)
	}
}
```

执行累计测试：

```bash
cd tutorials/02-multica-delivery/modules/02-runtime-access/v1/reference
go test ./...
```

预期 v1 和 v2 测试全部通过。这个版本冻结了 adapter 前的 access boundary，下一版只消费 `AccessVerdict`。

## 检查点与下一步

Runtime Access 现在能回答“这个成员能否使用这个 Runtime”和“Runtime 当前是否能执行”。v3 将把这两个 verdict 转换成 adapter input，确保拒绝、unknown 和 deferred 都不会启动外部进程。
