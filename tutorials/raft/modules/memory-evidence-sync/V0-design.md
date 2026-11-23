# Raft Memory / Evidence / Sync V0

## 版本与范围

- **源码**：`packages/daemon/src/agentContextView/*`、`packages/sync-core/src/*`、`packages/server/src/*`
- **目标**：解释 Agent context view、附件/证据收据、同步 domain 和 cursor。
- **排除**：完整知识图谱、Provider process 和所有 UI。

## 真实流程

```text
Agent activity/attachment → ContextView/EvidenceReceipt
  → server attachment/activity record
  → SyncDomain frame/snapshot/cursor
  → authorized Agent/client projection
```

Context view 是按 scope 投影给 Agent 的输入，不是数据库所有数据。EvidenceReceipt 记录来源和可核验引用，SyncCore 负责 domain frame 和恢复，不重新决定权限。

## 源码对应

| 路径 | 符号/责任 |
| --- | --- |
| `packages/daemon/src/agentContextView/*` | context source、restore、session summary 和脱敏投影 |
| `packages/sync-core/src/core.ts` | `createSyncCore`、domain scope、frame/cursor 生命周期 |
| `packages/sync-core/src/types.ts`、`domains/*` | 同步 domain 类型和状态 |
| `packages/server/src/*` | attachment/activity 持久化和权限查询 |

## 局部版本线

- V1：保存一条 evidence 和 context view。
- V2：附件引用、可见性和 Agent projection。
- V3：SyncDomain、snapshot/cursor 和离线重连。

## V1 检查

同一 evidence 在授权 Agent 可见，在无 scope Agent 不可见。同步断线后按 cursor 恢复不重复应用，记忆引用不能扩大执行权限。

契约见 `contracts/`，参考架构见 `reference-architecture.md`。
