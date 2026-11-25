# Multica Scheduler / Lease v1：claim 与 terminal update

[模块路线图](../plan.md) · [V0 设计](../V0-design.md) · [本版正文](claim-terminal.md) · [下一版 v2](../v2/plan.md)

## 版本定位

先按 job、scope 和计划时间建立 execution row。两个 worker 竞争同一计划，只有一个拿到 token。`Clock` 提供写入时间，并让后续版本可测试时间边界。

## 实现顺序

1. 定义 `Plan`、`Job`、`Execution` 与 `Clock`。
2. 实现 `TryClaim`、`FinishSuccess`、`FinishFailure`。
3. 测试两个 worker 竞争、计划唯一键与终态写入条件。

## 验收与交接

在 `v1/reference/` 运行 `go test ./...`。同一计划只有一个胜者，拿到 token 后可以写终态。V2 在同一行上增加 heartbeat 和 `StaleAfter`。
