# Event / Projection：参考架构

```text
EventSink → EventStore.append + ProjectionStore.apply
                    ↓
              CursorLog / Replay
                    ↓
              ThreadSubscription
```

参考实现文件为 `event_store.ts`、`event_sink.ts`、`projection_store.ts` 和 `cursor_subscription.ts`。V1 检查提交、读取、replay 和断线 cursor 恢复，并证明 replay 不调用 effect worker。
