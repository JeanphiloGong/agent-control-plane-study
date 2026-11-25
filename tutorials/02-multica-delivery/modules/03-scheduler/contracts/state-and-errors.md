# Scheduler / Lease：状态与错误

- 领取计划和比较过期时间使用 PostgreSQL 时间。心跳更新在 SQL 内计算新的期限。
- `RUNNING` 行超过 `stale_after` 后，允许重入且还有剩余尝试的 job 可以直接接管。`FAILED` 行的重试时间到达且仍有剩余尝试时可以重新领取。
- 每次重新领取生成新 token。旧 token 的心跳和终态提交都返回 `ErrLeaseLost`。
- `runHeartbeats` 收到 `ErrLeaseLost` 后退出心跳循环。handler 返回后的终态写入仍会检查 token。
