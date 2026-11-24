# Raft Daemon / Credential Boundary V0

## 版本与范围

- **源码**：`packages/daemon/src/agentHost/*`、`agentProcessManager.ts`、`packages/computer/src/*`、`agentCredentialProxy.ts`
- **目标**：解释 Server 指令如何到 Machine daemon、AgentHost、Provider process，并由 Credential Broker 代理凭据。
- **排除**：Channel/Task schema、长期记忆和同步。

## 真实流程

```text
server command → daemon/agentHost router
  → agentProcessManager start/stop
  → computer runtime
  → credential proxy scoped request
  → provider process / activity frame
```

Daemon 持有机器和进程生命周期，Server 不直接接触机器 secret。凭据代理依据 Agent/machine/capability scope 提供短期或受限访问。

## 源码对应

| 路径 | 符号/责任 |
| --- | --- |
| `packages/daemon/src/agentHost/*` | AgentHost router、启动、活动和结果摘要 |
| `packages/daemon/src/agentProcessManager.ts` | Agent process 生命周期、stall/restart 和 delivery |
| `packages/computer/src/*` | machine runtime/连接和本地能力 |
| `packages/daemon/src/agentCredentialProxy.ts` | credential proxy、scope 和 secret 注入边界 |

## 前置、产出与交接

- **前置**：[`01-identity-workspace`](../01-identity-workspace/V0-design.md) 的 capability scope 和 [`02-channel-task`](../02-channel-task/V0-design.md) 的 Task identity。
- **V1 产出**：daemon 注册、fake Agent process 生命周期和受 scope 限制的 credential request。
- **下游**：[`04-memory-evidence-sync`](../04-memory-evidence-sync/V0-design.md) 读取活动摘要和 evidence source。
- **冻结条件**：无机器 capability 时不能启动 process，Provider secret 不进入 Server message、Thread 或 Agent context。

## 局部版本线

- V1：daemon 注册并启动一个 fake Agent process。
- V2：stop/restart/stall 和 activity frame。
- V3：credential broker、scope 和 provider isolation。

## V1 检查

没有机器 capability 时拒绝启动。启动成功后读取活动摘要，模拟 process stall/restart。断言 secret 不出现在 server message、Thread 或 Agent context。

契约见 `contracts/`，参考架构见 `reference-architecture.md`。
