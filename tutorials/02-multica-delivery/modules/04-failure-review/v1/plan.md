# Multica Failure / Review v1：失败分类与 Review Gate

[模块路线图](../plan.md) · [V0 设计](../V0-design.md) · [本版正文](failure-review.md) · [下一版 v2](../v2/plan.md)

## 版本定位

本版为一次 Run 产生稳定的失败原因和策略决策。它区分“可以自动重试”和“需要 Review”，并保证成功 Run 进入 in_review，不会直接变成 shipped。

## 实现顺序

1. 定义 Run、FailureReason 和 Decision。
2. 实现 Classify 与 Decide。
3. 用网络、权限、业务失败和成功输入冻结边界。

## 验收与交接

在 v1/reference 运行 go test ./...。V2 复用 Decision.Retryable，加上 attempt budget、fire_at 和 Issue/workflow handoff。
