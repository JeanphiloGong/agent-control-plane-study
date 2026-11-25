# Multica Scheduler / Lease v3：过期接管与失败重试

[模块路线图](../plan.md) · [上一版 v2](../v2/plan.md) · [本版正文](stale-steal.md) · [下游 Failure / Review](../../04-failure-review/V0-design.md)

## 版本定位

在 V2 的执行行上增加两条重新领取路径。运行中的尝试要等 lease 超过期限且 job 允许重新进入。失败的尝试要等重试时间到达。两条路径都受 `MaxAttempts` 限制。

## 实现顺序

1. 给 `Job` 加 `AllowStaleReentry`，给 `Execution` 加 `NextRetryAt`。
2. 扩充 `TryClaim` 的冲突路径，保留 execution ID，递增 attempt，生成新 token。
3. 让 `FinishFailure` 保存调用方给出的重试时间，并更新 V1 的测试调用点。
4. 验收接管竞争、旧 token 拒绝、失败重试时间和尝试次数。

## 验收与交接

在 V1 的 Go module 执行 `go test ./...`。仍可沿着 `manager.go` 的 `processPlan`、`runClaimed` 追踪真实调度入口。下一个模块从已记录的失败结果继续讲错误分类、唤醒与 Review。
