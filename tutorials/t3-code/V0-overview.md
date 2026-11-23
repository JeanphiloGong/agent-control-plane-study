# T3 Code V0：完整 Agent 控制面

## 版本身份

- **模块**：`t3-code-control-plane`
- **基线**：`sources/t3code` @ `11224f63d2a3d444ca1fd1f67cd3ff21c1f5f8a1`
- **范围**：Command、Thread、Run、Provider Turn、Event、Projection、Effect Outbox、Workspace、Checkpoint 和 RPC Stream。
- **结果**：读者能理解服务端如何把用户命令、Provider 进程和客户端读模型拆成可追踪的事实流。

## 真实场景

客户端向一个 Environment 的 Thread 发送消息。Orchestrator 根据 CommandPolicy 选择启动、steer、fork 或拒绝，写入 Domain Event 和 Projection，再由 Effect Worker 在事务外启动 Provider Turn。Provider 事件重新进入服务端，客户端从 projection/stream 读取，不直接持有 Provider 状态。

```text
RPC command → Orchestrator / CommandPolicy
  → EventStore + ProjectionStore
  → EffectOutbox → EffectWorker → ProviderAdapter/Turn
  → provider result ingress → event/projection
  → ThreadStream / client runtime
```

## 模块

| 模块 | 责任 | V0 |
| --- | --- | --- |
| Command / Orchestrator | 命令判定、幂等和线程运行决策 | [orchestrator](modules/orchestrator/V0-design.md) |
| Event / Projection | 事实事件、投影、游标和重放 | [event-projection](modules/event-projection/V0-design.md) |
| Provider / Effects | Provider adapter、turn 生命周期和事务外副作用 | [provider-effects](modules/provider-effects/V0-design.md) |
| Workspace / Remote | Project、Worktree、Checkpoint、RPC 与订阅恢复 | [workspace-remote](modules/workspace-remote/V0-design.md) |

## 不变量

1. Command Receipt 和幂等键阻止同一命令重复创建 Run。
2. Event Store 的提交只记录服务端事实，外部 Provider 副作用由 outbox worker 执行。
3. Projection 是客户端读取模型，客户端不能越过服务端修改 Provider session。
4. Checkpoint 回退和远程断线都必须留下可查询的失败事实。

## 集成门 T1

一个 Provider Turn 要能从命令、事件、投影和副作用记录追踪到结果。数据库回滚、Provider 进程丢失和断线重连是必须保留的失败检查。
