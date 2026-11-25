# Multica Runtime Access v3：Runner adapter 启动与结果边界

[模块路线图](../plan.md) · [上一版 v2](../v2/plan.md) · [本版正文](runner-adapter.md)

## 版本定位

v2 已经得到 access 和 readiness verdict。v3 增加一个注入式 `RunnerAdapter`，只有 Allowed + Ready 才调用 `Start`，offline 返回 deferred，adapter failure 返回错误。Issue、Review 和 Scheduler 状态仍由各自模块拥有。

## 实现顺序

1. 定义 StartRequest、StartResult 和 RunnerAdapter。
2. 在 AccessService 中编排 authorize、deferred 和 adapter start。
3. 用 fake adapter 验证 ready 启动、offline 不启动、拒绝不启动和 adapter error。

## 验收与交接

累计 `go test ./...` 通过。下一模块从已注册系统任务的到期计划进入 claim、lease token 和 heartbeat。 [03 Scheduler / Lease](../../03-scheduler/V0-design.md)
