# Operation / Worktree：领域流

```text
parent operation → child item → session dispatch
  → watcher/materialization claim → result/deadline
  → parent aggregate

child session → allocated worktree → diff/review
```

父状态只能由 child 事实聚合，超时不等于成功。
