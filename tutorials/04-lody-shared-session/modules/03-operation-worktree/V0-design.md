# Lody Operation / Worktree V0

## 版本与范围

- **源码**：`coordinator-workflow.ts`、`operation-store.ts`、`session-dispatch-watcher.ts`、`session/worktree/*`
- **目标**：解释父操作如何派发 child session、跟踪 deadline、保存结果并隔离 worktree。
- **排除**：E2EE 和云同步。

## 真实流程

```text
CoordinatorWorkflow → OperationStore.accept
  → child items / session dispatch
  → dispatch watcher observes child
  → deadline/result/error materialization
  → parent operation completion
```

每个 child item 的状态独立于父 operation。`MATERIALIZATION_CLAIM_MS` 等 store 机制用于避免多个 watcher 同时物化同一结果。worktree 分配是写入隔离边界，不是 UI 标签。

## 源码对应

| 路径 | 符号/责任 |
| --- | --- |
| `coordinator-workflow.ts` | `CoordinatorWorkflow`、父子 operation 协调 |
| `operation-store.ts` | `LodyOperationStore`、fingerprint、materialization claim 和完成状态 |
| `session-dispatch-watcher.ts` | 观察 Session dispatch/child result |
| `session/worktree/*` | worktree 分配、路径和 diff/review 边界 |

## 前置、产出与交接

- **前置**：[`02-session`](../02-session/V0-design.md) 的 Session lifecycle 和 worktree ownership。
- **V1 产出**：一个 parent operation、child item、deadline 和隔离 worktree 的可查询结果。
- **下游**：[`04-sync-e2ee`](../04-sync-e2ee/V0-design.md) 同步 operation/document identity 和终态 cursor。
- **冻结条件**：child 超时、机器离线或 worktree 冲突都有明确结果，不能汇总成成功。

## 局部版本线

- V1：一个 Operation 派发一个 child。
- V2：多个 child、deadline、claim 和结果汇总。
- V3：worktree 隔离、diff 和 review feedback。

## V1 检查

一个父 operation 派发两个 child，完成一个、超时一个。断言父结果是部分/超时，而不是全完成。两个 child 的写入目录必须不同。

契约见 `contracts/`，参考架构见 `reference-architecture.md`。
