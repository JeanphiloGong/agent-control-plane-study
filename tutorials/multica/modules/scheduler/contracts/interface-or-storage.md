# Scheduler / Lease：接口契约

Execution row 至少需要 job id、status、attempt、lease token、heartbeat_at、stale_after 和 terminal result。`tryClaim` 返回 owner token，`heartbeat` 和 `finish*` 必须同时带 id 与 token。数据库更新的受影响行数是 lease 是否仍有效的事实。
