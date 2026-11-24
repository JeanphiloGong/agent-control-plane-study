# Raft Identity / Workspace V0

## 版本与范围

- **源码**：`packages/server`、`packages/shared`、`packages/raft-sdk`
- **目标**：解释 Workspace、Agent identity、机器绑定、presence 和 capability scope。
- **排除**：Channel/Task 交接、Daemon 进程和同步实现。

## 真实流程

```text
create workspace → create/register agent identity
  → membership + machine binding
  → capability scope / presence
  → server/SDK query
```

Agent 是长期参与者，不是某次 Task 的临时 worker。Workspace membership、机器绑定和 capability scope 必须分别保存，presence 只能表示可见的在线事实。

## 源码对应

| 路径 | 符号/责任 |
| --- | --- |
| `packages/server` | Workspace/Agent 领域服务、membership 和 presence API |
| `packages/shared` | Agent/Workspace 协议、schema 和跨端类型 |
| `packages/raft-sdk` | 人和 Agent 使用的 SDK 边界 |

## 前置、产出与交接

- **前置**：无。它定义 Raft 其他模块使用的身份、workspace 和 capability scope。
- **V1 产出**：创建 Agent、绑定 workspace/机器并读取 presence 的持久事实。
- **下游**：[`02-channel-task`](../02-channel-task/V0-design.md) 用 scope 判断消息可见性和 Task claim。
- **冻结条件**：Workspace membership、machine capability 和 Provider secret scope 分别判断。

## 局部版本线

- V1：创建 Agent、加入 Workspace、读取 capability。
- V2：机器绑定、presence 和 scope 更新。
- V3：与 Channel/Task、Daemon 权限交接。

## V1 检查

创建两个 Agent，分别加入不同 workspace。断言不能读取对方 workspace，且共享 workspace 不会共享 Provider secret。

契约见 `contracts/`，参考架构见 `reference-architecture.md`。
