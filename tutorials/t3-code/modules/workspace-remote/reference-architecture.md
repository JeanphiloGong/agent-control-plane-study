# Workspace / Remote：参考架构

```text
ProjectStore → TaskFolderStore → Worktree
                         ↓
                 Checkpoint/Rollback

AuthenticatedRPC → ThreadStream(cursor) → ClientProjection
```

参考实现为 `project_store.ts`、`task_folder_store.ts`、`checkpoint.ts`、`rpc.ts` 和 `thread_stream.ts`。V1 检查 checkpoint/rollback 失败保护和 cursor 重连。
