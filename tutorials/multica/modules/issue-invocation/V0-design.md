# Multica Issue / Agent Invocation V0

## 版本与范围

- **源码**：`sources/multica` @ `4ceffd536603c67620c7460f6c412dcbf092299e`
- **目标**：把 Issue 创建、成员授权、Agent readiness 和 Run 请求串成一条边界清楚的业务流。
- **排除**：Lease 实现、自动重试、完整客户端看板。

## 真实流程

```text
IssueService.Create
  → issue owner/member scope
  → CanMemberInvokeAgent
  → AgentReadiness
  → RuntimeLookup
  → RunRequest / task enqueue
```

Issue 是用户要求交付的事实。Agent 是被授权的执行主体，Runtime 是它可以使用的执行环境。Issue 创建成功不代表 Agent 已经启动，readiness 拒绝必须停在执行前。

## 源码对应

| 路径 | 符号/责任 |
| --- | --- |
| `server/internal/service/issue.go` | `IssueService`、`Create`，拥有 Issue 创建和工作区范围 |
| `server/internal/service/agent_invocation.go` | `CanMemberInvokeAgent`，判断成员能否调用 Agent |
| `server/internal/service/agent_ready.go` | `AgentVerdict`、`AgentReadiness`，分类 Runtime 不可用和修复建议 |
| `server/internal/service/agent_runtime_lookup.go` | `RuntimeLookup`，为 Issue/Task/Autopilot 服务提供 Runtime 查询 |
| `server/internal/service/agent_ready.go` | `RuntimeBlockedNeedsNotice`，把不可执行状态映射成用户可见阻塞 |

## 局部版本线

- V1：创建 Issue 并成功/拒绝一次 invocation。
- V2：绑定 Runtime 和 readiness 结果。
- V3：把 Run 结果回写 Issue。

## V1 检查

创建一个有权限且 Runtime ready 的 Issue，断言生成 Run 请求。对无权限 Agent、缺少 Runtime、离线 Runtime 各执行一次，断言没有启动副作用。

契约见 `contracts/`，参考架构见 `reference-architecture.md`。
