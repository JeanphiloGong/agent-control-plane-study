# Daemon / Credentials：参考架构

```text
ServerCommand → AgentHostRouter → AgentProcessManager → MachineRuntime
                                      ↓
                              CredentialBroker(scope)
                                      ↓
                                  ProviderProcess
```

学习实现为 `daemon.ts`、`agent_host.ts`、`process_manager.ts`、`computer_runtime.ts` 和 `credential_broker.ts`。V1 检查 capability denial、process lifecycle 和 secret non-observability。
