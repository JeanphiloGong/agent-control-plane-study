# Multica Issue / Invocation v2：Runtime readiness 与可等待队列

[模块路线图](../plan.md) · [上一版 v1](../v1/plan.md) · [本版正文](runtime-readiness.md) · [下一版 v3](../v3/plan.md)

## 版本定位

v1 只确认调用权限。v2 把 Agent 绑定的 Runtime 和 readiness 接入 invocation。online Runtime 产生立即可执行的 RunRequest，普通 offline Runtime 产生 `Deferred` 请求，缺少 Runtime、归档 Agent、权限不足或不可修复 Runtime 状态拒绝。

真实源码对应 `agent_ready.go::AgentReadiness`、`runtimeVerdict`、`RuntimeBlockedNeedsNotice` 和 `agent_runtime_lookup.go::RuntimeLookup`。参考项目使用内存 RuntimeRegistry，不实现数据库查询或 metrics。

## 实现顺序

1. 在 `v1/reference/issue/readiness.go` 定义 Runtime 状态、Availability、Reason 和 readiness verdict。
2. 扩展 RunRequest 的 runtime/deferred 字段，并替换 `Service.Create` 在权限通过后的 readiness 分支。
3. 测试 online、offline、missing runtime、archived Agent 和 unusable runtime。

## 验收与交接

累计 `go test ./...` 通过。重点检查 offline 只产生可等待请求，不启动外部进程；blocked verdict 不写 Issue 或 RunRequest。v3 在同一 RunRequest 上增加 runner 结果回写。
