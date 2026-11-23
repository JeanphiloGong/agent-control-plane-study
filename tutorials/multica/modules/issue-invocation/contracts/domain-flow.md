# Issue / Invocation：领域流

```text
Issue input → IssueService.Create → member/agent authorization
  → AgentReadiness → Runtime lookup → invocation/run request
```

Issue、Agent 和 Runtime 是三个不同生命周期。服务层负责把它们关联，不能用 Issue.status 代替 Agent readiness。
