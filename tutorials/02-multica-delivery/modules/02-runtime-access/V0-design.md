# Multica Runtime Access V0

## 版本与范围

- **源码**：`server/internal/service/runtime_access.go`、`agent_runtime_lookup.go`、`server/internal/runtimeapps/*`
- **目标**：解释 Runtime registry、访问策略、Agent host 和 Runner adapter 的边界。
- **排除**：数据库 Lease、跨区域调度和完整 Runtime 安装器。

## 真实流程

```text
Issue/Task service → RuntimeLookup → RuntimeAllowsUser
  → runtime status/ownership → runtimeapps adapter
  → start request or offline verdict
```

Runtime 是可被 Agent 使用的宿主，不是 Agent 本身。`RuntimeAllowsUser` 处理访问边界，lookup 为不同服务提供统一查询，runtimeapps 负责把宿主表示转成启动/控制调用。

## 源码对应

| 路径 | 符号/责任 |
| --- | --- |
| `server/internal/service/runtime_access.go` | `RuntimeAllowsUser`，判断用户是否能使用 Runtime |
| `server/internal/service/agent_runtime_lookup.go` | `RuntimeLookup`、各 Service 的 `runtimeLookup` |
| `server/internal/service/agent_ready.go` | `AgentReadiness` 读取 Runtime 状态并生成 verdict |
| `server/internal/runtimeapps/*` | Runtime app/runner adapter 的外部执行边界 |

## 前置、产出与交接

- **前置**：[`01-issue-invocation`](../01-issue-invocation/V0-design.md) 的成员/Agent 授权和 Run 请求。
- **V1 产出**：Runtime ownership、可用性和 runner start boundary 的统一 verdict。
- **下游**：[`03-scheduler`](../03-scheduler/V0-design.md) 只调度已经通过 access/readiness 的 Run；失败结果进入后续分类。
- **冻结条件**：越权或离线 Runtime 只返回拒绝，不创建进程，也不改变 Issue 的成功状态。

## 局部版本线

- V1：查询一个 Runtime 并判断可用性。
- V2：Agent 与 Runtime ownership/access policy。
- V3：Runner adapter 启动和结果回写。

## V1 检查

同一 Agent 在授权 Runtime 上可启动，在不属于 workspace 或离线 Runtime 上只返回拒绝，不创建进程。

契约见 `contracts/`，参考架构见 `reference-architecture.md`。
