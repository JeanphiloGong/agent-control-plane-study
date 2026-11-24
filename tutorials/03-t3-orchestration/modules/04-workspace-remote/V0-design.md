# T3 Workspace / Remote V0

## 版本与范围

- **源码**：`ProjectStore.ts`、`CheckpointService.ts`、`CheckpointRollbackService.ts`、`TaskFolderStore.ts`、`http.ts`、`ThreadStream.ts`、`packages/client-runtime/*`
- **目标**：解释 Project/Worktree/Checkpoint 与远程 RPC/订阅恢复的交接。
- **排除**：完整移动端和所有 Relay 部署。

## 真实流程

```text
Thread selects Project/TaskFolder
  → worktree change
  → CheckpointService.create
  → diff / rollback request
  → authenticated HTTP/RPC + ThreadStream
  → client projection and cursor resume
```

Worktree 是文件系统边界，Checkpoint 是可回退事实。远程客户端只发送命令和消费 stream，不能把本地猜测写成 server state。

## 源码对应

| 路径 | 符号/责任 |
| --- | --- |
| `ProjectStore.ts` | Project/workspace 持久化和查询 |
| `CheckpointService.ts` | 创建 checkpoint、记录文件状态 |
| `CheckpointRollbackService.ts` | 受控回退和失败保护 |
| `TaskFolderStore.ts` | Thread/task 到工作目录映射 |
| `orchestration-v2/http.ts` | 认证 RPC/HTTP 入口 |
| `ThreadStream.ts` | Thread 事件 stream、游标和重连 |
| `packages/client-runtime/*` | 客户端 runtime projection/transport |

## 前置、产出与交接

- **前置**：[`02-event-projection`](../02-event-projection/V0-design.md) 的客户端读取模型和 [`03-provider-effects`](../03-provider-effects/V0-design.md) 的 effect/result boundary。
- **V1 产出**：一个 Project/Worktree checkpoint 和断线后可恢复的订阅读取路径。
- **下游**：T1 追踪一个 Provider Turn 从命令、事件、投影到 workspace/effect 结果的完整链路。
- **冻结条件**：checkpoint 回退失败留下事实，远程断线不会把客户端本地状态当成服务端事实。

## 局部版本线

- V1：一个 Project/TaskFolder 和 checkpoint。
- V2：worktree diff、rollback 和部分失败。
- V3：远程 RPC、stream cursor 和重连。

## V1 检查

在临时 worktree 创建 checkpoint，修改后成功回退。模拟文件回退失败，断言 checkpoint 状态不是伪成功。断开 stream 后从 cursor 恢复。

契约见 `contracts/`，参考架构见 `reference-architecture.md`。
