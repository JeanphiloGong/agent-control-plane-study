# Scheduler / Lease：状态与错误

- `RUNNING` 只有当前 lease token 可以 heartbeat 或结束。
- 领取和 stale 判断使用数据库 `now()`，不能混用 worker 本地时间。
- heartbeat 返回 `ErrLeaseLost` 时 runner 应停止继续写入。
- stale steal 产生新 token，旧 token 的终态更新被忽略或拒绝。
- finish 失败不能把执行报告成成功。
