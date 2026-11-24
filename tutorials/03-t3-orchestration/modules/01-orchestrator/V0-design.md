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

## 前置、产出与交接

- **前置**：无。它定义客户端命令进入控制面的唯一决策入口。
- **V1 产出**：一个带幂等 receipt 的 command decision，以及线程/run 的启动、steer、fork 或拒绝结果。
- **下游**：[`02-event-projection`](../02-event-projection/V0-design.md) 持久化 decision event 和 projection；Provider 不直接读取命令请求。
- **冻结条件**：相同幂等键不会重复创建 Run，拒绝命令不会产生 Provider 或 workspace 副作用。

## 局部版本线

- V1：一个 Thread 启动一个 Run。
- V2：Command Receipt 和幂等命令。
- V3：steer、fork、rollback capability policy。

## V1 检查

对相同 command id 重复提交，断言只得到一个 receipt/run。对无 active run 的 steer、不可 native fork 的 provider 和不支持 rollback 的状态，断言策略拒绝且不产生外部效果。

契约见 `contracts/`，参考架构见 `reference-architecture.md`。
