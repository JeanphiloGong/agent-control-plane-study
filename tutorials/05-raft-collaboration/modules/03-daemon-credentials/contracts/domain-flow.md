# Daemon / Credentials：领域流

```text
Server capability request → Daemon/AgentHost → ProcessManager
  → scoped CredentialBroker → Provider process
  → activity/result frame → Server
```

Daemon 是机器边界，CredentialBroker 是 secret 边界，AgentHost 是进程编排边界。
