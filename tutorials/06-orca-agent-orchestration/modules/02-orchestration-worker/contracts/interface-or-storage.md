# Orca Orchestration / Worker：接口与存储契约

## 输入

- Task ID 或完整 spec
- owning Run、worker identity、agent 和 execution host
- worktree placement、repo、base branch 和 setup choice
- 可选 retry predecessor、timeout 和 startup preferences

## 输出

- Worker start receipt，包含 Dispatch ID、Task ID、阶段、状态、错误和 next action
- Worker projection，包含 liveness、workspace、terminal resource 和 last report
- Settlement result，包含当前 Dispatch status、Task status 和可继续执行的 next action

## 存储边界

- Task 保存任务目标和 Run 归属。
- Dispatch 保存一次 attempt 的身份、predecessor 和 worker settlement。
- Workspace 保存执行目录与 host identity。
- Receipt 是 mutation response，可以丢失并通过 worker-show / task-list 重新读取。
- 旧 Dispatch 不能因为新 receipt 被删除，恢复和审计需要完整尝试链。

## 禁止的耦合

- CLI view 不能直接改变 Task status。
- Receipt 不能替代 Dispatch post-verification。
- Worker report 不能绕过 owning Run 直接标记 Review 或 merge。
- 一个 Task 的新 retry 不能复用旧 Dispatch ID。
