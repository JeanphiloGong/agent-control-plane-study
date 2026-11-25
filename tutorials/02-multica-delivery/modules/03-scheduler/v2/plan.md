# Multica Scheduler / Lease v2：心跳续期

[模块路线图](../plan.md) · [上一版 v1](../v1/plan.md) · [本版正文](heartbeat.md) · [下一版 v3](../v3/plan.md)

## 版本定位

V1 的 runner 能领取并提交终态。执行可能持续几分钟，V2 让同一 token 在运行期间更新 `HeartbeatAt` 和 `StaleAfter`，为 V3 的接管判断提供依据。

## 实现顺序

1. 在 `Job` 增加 `StaleTimeout`，领取时写入心跳和过期时间。
2. 添加 `Store.Heartbeat`，以 execution ID、token 和 `RUNNING` 为写入条件。
3. 用 `FixedClock` 推进时间，检查当前持有者续期、错误 token 被拒绝和终态停止续期。

## 验收与交接

在 V1 的 Go module 执行 `go test ./...`。固定时钟推进后，当前 lease 的过期时间向后移动。V3 在同一行上使用过期时间决定是否接管。
