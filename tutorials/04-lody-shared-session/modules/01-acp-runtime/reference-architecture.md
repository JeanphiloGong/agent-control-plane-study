# ACP Runtime：参考架构

```text
MachineRuntime → AcpLauncher → AcpTransport → AgentClient
       ↓                                      ↓
  install/status                       SessionEvent adapter
```

参考实现为 `machine_runtime.ts`、`acp_launcher.ts`、`agent_client.ts` 和 `session_events.ts`。V1 检查 fake process 的启动、event 转换和分类失败。
