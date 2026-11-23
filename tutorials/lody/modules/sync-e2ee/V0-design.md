# Lody Sync / E2EE V0

## 版本与范围

- **源码**：`packages/shared/src/session-data/*`、`session-doc-schemas.ts`、`packages/e2ee-core/*`、`packages/loro-streams-rpc/*`
- **目标**：区分 Session 文档、同步域、设备密钥、传输帧和离线冲突。
- **排除**：完整云服务、Provider runtime 和 Operation workflow。

## 真实流程

```text
local session/document mutation
  → schema/domain frame
  → encrypt/sign with device key
  → Loro/RPC transport
  → remote apply/snapshot/cursor
  → conflict/offline reconciliation
```

同步传播的是声明过 schema 的文档/事件。设备密钥和 Agent credential 不进入普通 Session document。离线恢复需要 cursor/idempotency，不能简单覆盖本地状态。

## 源码对应

| 路径 | 符号/责任 |
| --- | --- |
| `packages/shared/src/session-data/*` | Session/Workspace/Message 数据模型 |
| `packages/shared/src/session-doc-schemas.ts` | 文档 schema 和输入边界 |
| `packages/e2ee-core/*` | device key、加密和密钥生命周期 |
| `packages/loro-streams-rpc/*` | Loro stream、RPC frame 和同步 transport |

## 局部版本线

- V1：本地 Session document 和 schema。
- V2：domain frame、cursor、离线重放和冲突。
- V3：E2EE key boundary 和多设备恢复。

## V1 检查

两个本地副本产生可合并文档变化，断线后按 cursor 重放不重复。无 key 或错误 key 时拒绝解密，不把密文当作成功状态。

契约见 `contracts/`，参考架构见 `reference-architecture.md`。
