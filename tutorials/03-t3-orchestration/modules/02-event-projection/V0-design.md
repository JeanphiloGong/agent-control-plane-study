# T3 Event / Projection V0

## 版本与范围

- **源码**：`EventStore.ts`、`EventSink.ts`、`ProjectionStore.ts`、`packages/contracts/src/orchestrationV2.ts`
- **目标**：解释服务端事实事件、事务提交、投影读模型和订阅游标。
- **排除**：Provider adapter 和 outbox worker 的实际执行。

## 真实流程

```text
accepted command → EventSink
  → EventStore append + ProjectionStore update
  → cursor/stream subscription
  → client reads projection
```

EventStore 是事实日志，ProjectionStore 是查询模型。事件提交和投影更新的事务边界必须在 V0 里明确，客户端断线后用游标重新读取服务端事实。

## 源码对应

| 路径 | 符号/责任 |
| --- | --- |
| `EventStore.ts` | 事件持久化和读取 |
| `EventSink.ts` | 事务内写事件并更新 projection |
| `ProjectionStore.ts` | Thread/Run 读模型和游标读取 |
| `packages/contracts/src/orchestrationV2.ts` | Event、Projection、Run/Thread schema |

## 局部版本线

- V1：append event 并读取当前 projection。
- V2：事务内 event/projection、cursor 和 replay。
- V3：多客户端订阅和投影重建。

## V1 检查

追加一个 run-started 事件，断言 projection 更新。重放事件得到同一 projection，重复 cursor 不重复消费。

契约见 `contracts/`，参考架构见 `reference-architecture.md`。
