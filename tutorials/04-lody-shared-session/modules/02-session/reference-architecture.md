# Session Lifecycle：参考架构

```text
SessionCommand → SessionManager → SessionExecutionService
                                      ↓
                               AgentRuntime adapter
                                      ↓
                          SessionEvent / SessionStore
```

参考实现为 `session_store.ts`、`session_manager.ts`、`execution_service.ts` 和 `control_api.ts`。V1 检查 session 状态、turn 完成、cancel outcome 和 resume 前置条件。
