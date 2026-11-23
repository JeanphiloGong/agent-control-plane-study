# Runtime Access：参考架构

```text
RuntimeRegistry → AccessPolicy → ReadinessService → RunnerAdapter
       ↑                ↑                 ↓              ↓
   runtime record   member/owner       verdict       external process
```

学习实现放在 `runtime_registry.go`、`access_policy.go`、`readiness.go` 和 `runner_adapter.go`。参考实现用 fake runner 验证 access policy 不会被绕过。

对应源码为 `RuntimeAllowsUser`、`RuntimeLookup` 和 `server/internal/runtimeapps/*`。V1 检查包含 ready 启动、offline 拒绝和跨 workspace 拒绝。
