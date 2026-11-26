# Multica Failure / Review v2：有限重试与 Issue workflow handoff

[模块路线图](../plan.md) · [上一版 v1](../v1/plan.md) · [本版正文](retry-policy.md) · [下一版 v3](../v3/plan.md)

## 版本定位

本版把 V1 的 Retryable 变成可执行的 retry child 计划。计划拥有尝试次数、等待时间和原 Issue/workflow identity。自动重试不创建无限循环，也不丢失交接 brief。

## 实现顺序

1. 使用 RetryPlan 和 RetryState 表达 queued/deferred。
2. 为 provider network 设置有效次数上限和最终延迟。
3. 保留 Issue、workflow step 和 handoff，并测试权限失败与预算耗尽。

## 验收与交接

在 v1/reference 运行 go test ./...。V3 复用 RetryPlan.FireAt，将 deferred child 交给 wakeup 入口。
