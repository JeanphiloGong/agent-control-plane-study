# T3 Command / Orchestrator V0

## 版本与范围

- **源码**：`apps/server/src/orchestration-v2/Orchestrator.ts`、`CommandPolicy.ts`、`ThreadCommandExecutor.ts`
- **目标**：把客户端命令转成一个可判定、可幂等、可进入事件流的线程/运行决策。
- **排除**：Provider 进程、完整 Event Store 和远程 RPC。

## 真实流程

```text
client command → CommandPolicyV2
  → resolve delivery/steer/fork/rollback intent
  → ThreadCommandExecutor
  → Orchestrator
  → CommandReceipt + DomainEvent
```

`CommandPolicyV2` 依据 projection、active run 和 provider capability 决定命令是否支持。`ThreadCommandExecutor` 负责串行化敏感命令，Orchestrator 负责把决策交给事实/副作用边界。

## 源码对应

| 路径 | 符号/责任 |
| --- | --- |
| `Orchestrator.ts` | `Orchestrator`，接收命令并编排线程/运行决策 |
| `CommandPolicy.ts` | `CommandPolicyV2`、`resolveMessageDispatchIntent`、fork/steer/rollback checks |
| `ThreadCommandExecutor.ts` | `ThreadCommandExecutor` 和命令串行化 |
| `packages/contracts/src/orchestrationV2.ts` | command、run、thread、receipt 的 wire schema |

## 局部版本线

- V1：一个 Thread 启动一个 Run。
- V2：Command Receipt 和幂等命令。
- V3：steer、fork、rollback capability policy。

## V1 检查

对相同 command id 重复提交，断言只得到一个 receipt/run。对无 active run 的 steer、不可 native fork 的 provider 和不支持 rollback 的状态，断言策略拒绝且不产生外部效果。

契约见 `contracts/`，参考架构见 `reference-architecture.md`。
