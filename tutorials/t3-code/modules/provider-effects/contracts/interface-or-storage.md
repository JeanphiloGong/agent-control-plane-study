# Provider / Effects：接口契约

`ProviderAdapter` 输入统一的 thread/turn request，输出事件流或明确错误。`PendingEffect` 包含 effect id、request type、source run 和尝试信息。Worker 只消费已提交 effect，并通过 result ingress 回到 EventSink。
