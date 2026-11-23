# T3 Provider / Effect Outbox V0

## 版本与范围

- **源码**：`ProviderAdapterRegistry.ts`、`ProviderSessionManager.ts`、`ProviderTurnStartService.ts`、`EffectOutbox.ts`、`EffectWorker.ts`
- **目标**：把 Provider 进程和外部副作用放到事务外，并把结果重新送回编排。
- **排除**：所有 Provider 的完整适配和多端 UI。

## 真实流程

```text
committed run-start event → EffectOutbox
  → EffectWorker
  → ProviderAdapterRegistry / ProviderSessionManager
  → Provider turn events
  → result ingress → EventSink/Projection
```

Provider session 是外部进程生命周期，Run 是服务端事实生命周期。数据库事务回滚时，不能声称 Provider 已运行。worker 重试也必须处理已启动、已完成和不可重试的窗口。

## 源码对应

| 路径 | 符号/责任 |
| --- | --- |
| `ProviderAdapterRegistry.ts` | provider 名称到 adapter 的注册和选择 |
| `ProviderSessionManager.ts` | Provider session 的创建、查找和关闭 |
| `ProviderTurnStartService.ts` | 依据 run/thread 投递 turn |
| `EffectOutbox.ts` | 已提交事实对应的 pending effect |
| `EffectWorker.ts` | 消费 effect、调用 provider、处理重试和结果回写 |

## 局部版本线

- V1：fake Provider adapter 完成一个 turn。
- V2：outbox 与 worker，区分 committed/pending/running/failed。
- V3：真实 Provider event ingress 和进程丢失处理。

## V1 检查

事务提交后 effect 才可执行。模拟事务回滚、worker 重启和 Provider 进程退出，断言没有假成功 Run。

契约见 `contracts/`，参考架构见 `reference-architecture.md`。
