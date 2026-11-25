# Scheduler / Lease：接口契约

`(job_name, scope_kind, scope_id, plan_time)` 是执行行的唯一键。行内保存 `status`、`attempt`、`max_attempts`、`runner_id`、`lease_token`、`heartbeat_at`、`stale_after`、`next_retry_at` 和终态结果。

`tryClaim` 在首次领取或重新领取成功时返回 execution ID、token 和 attempt。`heartbeat`、`finishSuccess` 和 `finishFailure` 同时匹配 ID、token、`RUNNING`。更新影响零行时返回 `ErrLeaseLost`。
