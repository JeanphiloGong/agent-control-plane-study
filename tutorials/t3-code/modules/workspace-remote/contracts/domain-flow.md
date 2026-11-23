# Workspace / Remote：领域流

```text
Thread → Project/TaskFolder → Worktree mutation → Checkpoint
  → rollback/diff → server fact → RPC/ThreadStream → client projection
```

文件系统操作和远程订阅都必须回到服务端拥有的 Project/Thread 事实。
