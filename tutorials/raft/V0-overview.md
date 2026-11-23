# Raft V0：人和 Agent 的共享工作空间

## 版本身份

- **模块**：`raft-collaboration-space`
- **基线**：`sources/raft-source` @ `5214e34e726ed715f455e78c96fa521791ef2de3`
- **范围**：Workspace、Agent Identity、Channel、Thread、Task、Daemon、Credential Broker、Evidence 和 Sync Domain。
- **结果**：读者能理解长期 Agent 参与者如何在共享空间接收任务、使用机器能力并回传证据。

## 真实场景

一个 Agent 加入 Workspace，在 Channel 中看到自己有权限的任务。Server 负责消息和任务事实，Daemon 在机器上启动 Agent，Credential Broker 只在能力授权后代理 Provider secret。Agent 把执行摘要和证据附件回传到 Thread。同步层把领域事件送到其他设备，但不扩大消息、任务或机器权限。

```text
AgentIdentity + Workspace membership
  → Channel/Thread visibility
  → Task claim
  → Server → Daemon/AgentHost → Provider process
  → CredentialBroker scoped access
  → Evidence/Attachment → SyncDomain → other clients
```

## 模块

| 模块 | 责任 | V0 |
| --- | --- | --- |
| Identity / Workspace | Agent 身份、机器绑定、presence 和能力范围 | [identity-workspace](modules/identity-workspace/V0-design.md) |
| Channel / Task | 消息、Thread、Task 和可见性交接 | [channel-task](modules/channel-task/V0-design.md) |
| Daemon / Credentials | 机器 daemon、AgentHost、进程和凭据代理 | [daemon-credentials](modules/daemon-credentials/V0-design.md) |
| Memory / Evidence / Sync | Context view、附件、证据收据和实时同步 | [memory-evidence-sync](modules/memory-evidence-sync/V0-design.md) |

## 不变量

1. Workspace membership 不等于机器权限，也不等于 Provider secret。
2. 消息可见性、Task claim 权限、机器 capability 和凭据范围分别判断。
3. Evidence 是可引用的事实，不自动扩大 Agent 的执行权限。
4. Sync frame 传播状态，不绕过 Server 的授权边界。

## 集成门 R1

有权限的 Agent 从 Channel 领取 Task，经 Daemon 执行并回传证据。必须证明无权限 Agent 看不到任务、无机器能力不能启动 Provider、secret 不进入消息或模型上下文。
