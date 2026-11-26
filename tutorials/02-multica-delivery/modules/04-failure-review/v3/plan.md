# Multica Failure / Review v3：Issue Wakeup 与客户端状态投影

[模块路线图](../plan.md) · [上一版 v2](../v2/plan.md) · [本版正文](wakeup-projection.md)

## 版本定位

本版连接 deferred retry 与 Issue Wakeup，并把服务端决策转换为客户端可显示的状态。Wakeup 条件不满足时保持等待，已经 dispatch 的一次性规则不能再次触发。

## 实现顺序

1. 检查 Wakeup 的启用状态、到期时间和 Issue 条件。
2. 记录一次性 dispatch，生成保留原 Issue identity 的 Run。
3. 将 Run Decision 投影为 in_progress、blocked 或 in_review，不把成功直接当成 shipped。

## 验收

在 v1/reference 运行 go test ./... 和 go test -race ./...。真实服务集成测试路径见 issue_wakeup_*_test.go，需要独立 PostgreSQL fixture。
