# Issue / Invocation：参考架构

```text
IssueCommand → IssueService → AuthorizationPolicy
                         → ReadinessService → RuntimeRegistry
                         → RunRequest
```

学习实现文件建议为 `issue_service.go`、`invocation_policy.go`、`readiness.go` 和 `run_request.go`。IssueService 只编排，AuthorizationPolicy 不启动进程，Runner 只消费已经通过的 RunRequest。

源码映射：`IssueService.Create` 对应 `IssueService.create`，`CanMemberInvokeAgent` 对应 `InvocationPolicy.allow`，`AgentReadiness` 对应 `ReadinessService.check`。

V1 冻结检查是 ready invocation 成功，三个拒绝场景没有 Runner 调用。
