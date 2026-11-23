# Command / Orchestrator：领域流

```text
Command → Policy decision → Executor serialization → Orchestrator
  → receipt + domain event → later effect/projection
```

Policy 只决定是否允许和采用哪种 execution mode，Orchestrator 不把客户端意图直接当作 Provider 状态。
