# Lody V0：ACP 多机器共享会话

## 版本身份

- **模块**：`lody-acp-workspace`
- **基线**：`sources/lody` @ `762dee0431ecae0649b9fab13a1f357b096d0ac`
- **范围**：Machine、ACP Agent Runtime、Session、Operation、Worktree、Sync/E2EE。
- **结果**：读者能区分 Agent 进程、会话、跨会话操作和同步文档的生命周期。

## 真实场景

一台机器上的 Lody daemon 通过 ACP 启动 Codex 或 Claude session。用户可以读取、steer、cancel、archive 和 resume。协调器把一个父操作拆到多个 Session，operation store 记录子项和截止时间。每个 Session 使用自己的工作树。同步层传播文档和状态，但不把设备密钥或 Provider 凭据当作会话消息。

```text
Machine → ACP runner/client → SessionManager/ExecutionService
  → CoordinatorWorkflow → OperationStore → child Session
  → WorktreeAllocator / Diff / Review
  → SyncDocument / SyncDomain / DeviceKey transport
```

## 模块

| 模块 | 责任 | V0 |
| --- | --- | --- |
| 01 ACP Runtime | 机器上的 Agent 进程和 ACP capability | [01-acp-runtime](modules/01-acp-runtime/V0-design.md) |
| 02 Session | 会话创建、执行、控制和恢复 | [02-session](modules/02-session/V0-design.md) |
| 03 Operation / Worktree | 跨会话派发、截止时间和隔离目录 | [03-operation-worktree](modules/03-operation-worktree/V0-design.md) |
| 04 Sync / E2EE | Session 文档、同步帧、设备密钥和离线边界 | [04-sync-e2ee](modules/04-sync-e2ee/V0-design.md) |

## 模块依赖

```text
01 ACP Runtime
  → 02 Session
  → 03 Operation / Worktree
  → 04 Sync / E2EE
  → L1
```

`01` 先建立机器上的 ACP 进程和统一事件边界。`02` 拥有 Session 生命周期，不能把
Provider 进程状态当成 Session 状态。`03` 使用 Session 派发 child operation，并为每个
Session 分配工作树。`04` 最后同步文档和游标，不能绕过 Session 或 Operation 的权限。

## 不变量

1. ACP provider 细节由 AgentClient/adapter 隔离，Session 上层只消费统一事件。
2. Session 状态和 Operation 状态分别建模，不能都压成一个 `Task.status`。
3. Operation 超时或 child 不可读时不能伪造完成。
4. Worktree 是 Session 的写入边界，平行 Session 不能共享未授权目录。

## 集成门 L1

一个父 Session 派发一个 child Operation，能分别查询父子状态并汇总结果。机器离线、child 超时和工作树冲突必须有显式结果。
