# Command / Orchestrator：参考架构

```text
CommandGateway → CommandPolicy → ThreadCommandExecutor → Orchestrator
                                      ↓                     ↓
                                serialized lane       Receipt/Event
```

参考实现使用 `command.ts`、`policy.ts`、`executor.ts` 和 `orchestrator.ts`。executor 不访问 Provider，Orchestrator 不直接给客户端发“完成”消息。

V1 冻结检查是重复 command id 幂等和 unsupported capability 无副作用。
