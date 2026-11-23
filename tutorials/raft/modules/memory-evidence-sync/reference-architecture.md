# Memory / Evidence / Sync：参考架构

```text
ContextViewSource → RedactedContextView → EvidenceStore
                                      ↓
                               SyncDomain → FrameTransport
                                      ↓
                              Snapshot/Cursor projection
```

学习实现为 `context_view.ts`、`evidence_store.ts`、`sync_core.ts`、`domain_frame.ts` 和 `cursor.ts`。V1 检查 scope 过滤、证据引用、重复 frame 和断线 cursor 恢复。
