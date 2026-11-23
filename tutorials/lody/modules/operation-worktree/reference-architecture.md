# Operation / Worktree：参考架构

```text
Coordinator → OperationStore → DispatchWatcher → ChildSession
                     ↓                              ↓
              deadline/claim                  WorktreeAllocator
                     └──────── result/review ───────┘
```

学习实现为 `operation_store.ts`、`coordinator.ts`、`dispatch_watcher.ts` 和 `worktree_allocator.ts`。V1 检查两个 child 的独立状态、过期 deadline 和目录隔离。
