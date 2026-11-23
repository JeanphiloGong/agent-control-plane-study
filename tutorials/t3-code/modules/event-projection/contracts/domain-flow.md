# Event / Projection：领域流

```text
domain fact → EventSink transaction → EventStore + ProjectionStore
  → cursor subscription → client read model
```

Projection 可以重建，Event 是事实。客户端不把自己的缓存当作 authoritative state。
