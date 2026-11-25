# Multica Issue / Invocation v1：权限通过后生成 Run 请求

[模块路线图](../plan.md) · [V0 设计](../V0-design.md) · [本版正文](issue-invocation.md) · [下一版 v2](../v2/plan.md)

## 版本定位

先把 issue 创建和 agent invocation 权限放在同一个服务边界。成功路径写入 Issue 并返回 RunRequest，拒绝路径在写入 Issue 前结束。Runtime 状态暂由 v2 接入。

## 实现顺序

1. 在 `v1/reference/issue/model.go` 定义 Issue、Agent、Member、InvocationTarget 和 RunRequest。
2. 在 `v1/reference/issue/policy.go` 实现 owner、workspace target、member target 和 guest 规则。
3. 在 `v1/reference/issue/service.go` 编排校验、Issue 写入和 RunRequest 生成。

## 验收与交接

`go test ./...` 通过。验收覆盖 owner、workspace target、guest 和无权限路径，并断言拒绝不会写入 Issue。v2 保留同一 RunRequest，加入 Runtime readiness。
