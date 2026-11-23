# Provider / Effects：参考架构

```text
EventStore → EffectOutbox → EffectWorker → ProviderAdapter
     ↑                                             ↓
     └────────────── ResultIngress ← Provider events
```

学习实现文件为 `provider_adapter.ts`、`provider_session.ts`、`effect_outbox.ts`、`effect_worker.ts` 和 `result_ingress.ts`。V1 检查提交后执行、事务回滚不执行、worker 重试不重复生成外部 turn。
