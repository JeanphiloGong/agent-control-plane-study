# Raft Channel / Thread / Task V0

## 版本与范围

- **源码**：`packages/server`、`packages/shared`、`manual/recipes/*`
- **目标**：解释消息、Thread、Task、visibility policy 和交接。
- **排除**：Daemon 启动、凭据代理和同步冲突。

## 真实流程

```text
message/task command → workspace/channel visibility
  → Thread record → Task create/claim
  → handoff message/evidence reference
```

消息可见性和 Task claim 权限不是同一个判断。Thread 记录讨论上下文，Task 记录可执行交付，Agent 只能领取其 scope 允许的 Task。

## 源码对应

| 路径 | 责任 |
| --- | --- |
| `packages/server` | Channel/Thread/Task 服务、权限和 realtime 事实 |
| `packages/shared` | message/session/task 协议与 schema |
| `manual/recipes/*` | Agent 使用 Channel/Task 的 CLI/SDK 交接示例 |

## 局部版本线

- V1：Channel/Thread 消息读取和发送。
- V2：Task 创建、claim、reply 和 visibility policy。
- V3：证据附件、多人/多 Agent handoff 和 realtime。

## V1 检查

两个 Agent 位于不同 scope。断言不可见消息和无权限 Task claim 都被拒绝，有权限 Agent 可以领取并回传一条 evidence reference。

契约见 `contracts/`，参考架构见 `reference-architecture.md`。
