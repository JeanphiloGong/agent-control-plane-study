# Sync / E2EE：参考架构

```text
SessionDocument → Schema → SyncDomain → FrameCodec → E2EE → Transport
       ↑                                                        ↓
       └────────────── apply/cursor/conflict ───────────────────┘
```

学习实现为 `document_schema.ts`、`sync_domain.ts`、`device_key.ts`、`frame_transport.ts`。V1 检查 schema、cursor 幂等、离线恢复和错误 key 拒绝。
