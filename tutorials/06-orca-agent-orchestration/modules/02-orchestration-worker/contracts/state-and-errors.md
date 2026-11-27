# Orca Orchestration / Worker：状态与错误契约

## 状态

~~~text
Task ready
  → Dispatch created
  → workspace-created
  → session-ready
  → turn-started
  → succeeded / failed / stopped / unknown
  → Task running / succeeded / failed
~~~

## 错误边界

- worktree create failure 保留失败 stage。
- session-not-ready 拒绝 turn start。
- start_unknown 表示 mutation 结果未知，不表示 worker 已退出。
- operation_unknown 要求读取 Dispatch 和 Task，而不是重新发送一份新 mutation。
- 重复的完全相同 report 返回幂等成功，冲突 report 返回 settled conflict。
- Dispatch 的失败不自动清理 workspace。Recovery 和 Worktree cleanup 分别拥有这两个决定。

## 必测状态

1. 同一个 Task 下两个 worker 可以分别创建 Dispatch。
2. 同一个 worker 的第二次 start 必须显式说明 retry 或新 Task。
3. session ID 改变时拒绝原 Dispatch 的观察。
4. 旧 Dispatch report 不能结算当前 Dispatch。
