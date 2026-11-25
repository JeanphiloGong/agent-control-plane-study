# Multica Issue / Invocation v3：Run 结果回写 Issue

[模块路线图](../plan.md) · [上一版 v2](../v2/plan.md) · [本版正文](run-result.md)

## 版本定位

v2 已经生成带 Runtime 和 deferred 语义的 RunRequest。v3 增加一个注入式 Runner 边界，让 Run 从 queued 进入 running，再把成功或失败结果回写 Issue。Review 状态保持 pending，后续 Failure / Review 模块负责分类、重试和审核。

## 实现顺序

1. 扩展 Issue 和 RunRequest 的状态字段。
2. 在 `service.go` 增加 `StartRun`、`CompleteRun` 和 `Execute`，用状态检查拒绝重复完成。
3. 在 `runner.go` 定义 fake runner，验证成功、失败和 review 独立性。

## 验收与交接

累计 `go test ./...` 通过。成功 Run 只能把 Issue 标成 done，不能把 review 标成 approved；失败 Run 把 Issue 标成 blocked；非 running Run 不能二次写终态。下一个 `02-runtime-access` 模块会把 Runtime 查找和实际 runner adapter 拆出来。
